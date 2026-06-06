# Flutter 작성 시트 자동저장 패턴 — Mixin 템플릿메서드 + Debounce + 로컬 드래프트

> 한 줄 요약: "타이핑할 때마다 일정 시간 기다렸다 서버에 자동저장 + 로컬에 백업"하는 공통 로직을, `mixin`(빈칸 채우기 계약)으로 추출해 여러 작성 화면이 재사용하는 패턴.

## 핵심
- **Mixin = 템플릿 메서드 패턴.** "언제 저장할지(공통 흐름)"는 mixin이 들고, "어떻게 저장할지(구체 행동)"는 화면이 `@override`로 채운다. `class X extends State with AutoSaveControllerMixin` 처럼 `with`로 부품을 장착하면 mixin의 메서드를 내 것처럼 쓴다. (`extends`=부모 1개, `with`=기능 부품 여러 개)
- **Debounce(디바운스).** 글자 칠 때마다 `timer.cancel()` 후 `Timer(delay, save)`를 다시 건다. 타이핑을 멈춰 delay 동안 조용해진 순간 딱 한 번만 실제 저장이 돈다. → 매 글자 서버 호출(N번)을 1번으로 합침.
- **이중 저장소.** ① 서버(API, 0.6s debounce) = 진짜 영구 데이터. ② 로컬(SharedPreferences, 0.3s debounce) = 앱이 죽어도 살아남는 임시 안전망(드래프트). 로컬이 더 싸니 더 자주 백업.
- **드래프트 의미론:** 작성(create) 모드에서만 저장 / **정상 close 시 삭제** / **강제종료 시 살아남아** 재오픈 때 복원 / 7일 경과 자동폐기.
- **flush-on-close.** 닫기 직전, debounce가 아직 안 끝나 미저장된 내용이 날아가지 않도록 `flushPendingSave()`로 마지막 저장을 강제 await한 뒤 화면을 닫는다.

## 언제 / 왜 쓰나
- 메모/주소증/진료기록처럼 **"저장 버튼 없이 알아서 저장"** 되는 작성 UI가 여러 개일 때.
- 같은 자동저장 동작을 화면마다 복붙하지 않고 한 곳(mixin)에 모아 일관성·테스트성을 확보.
- 모바일 특성상 앱이 OS에 의해 백그라운드에서 강제종료될 수 있으므로, 입력 유실 방지용 로컬 백업이 필요할 때.

## 예시 (코드·명령)
실제 구현: `lib/features/consultation/presentation/widgets/write_sheet/auto_save_controller.dart` (mixin) +
`.../jusojung/write_jusojung_sheet.dart` (구현체)

```dart
// 1) mixin은 "빈칸"만 선언 (abstract)
mixin AutoSaveControllerMixin<T extends StatefulWidget, R> on State<T> {
  // 화면이 채워야 할 계약
  R buildRecord();                       // 폼값 → 모델
  Future<R> performCreate(R r);          // 생성 API
  Future<R> performUpdate(R r);          // 수정 API
  Future<void> performDelete(R r);       // 삭제
  bool isContentEmpty();                 // 빈 내용?
  void invalidateList();                 // 저장 후 리스트 갱신
  Map<String,dynamic> serializeDraft();  // 폼 → Map (로컬백업)
  void applyDraft(Map<String,dynamic>);  // Map → 폼 (복원)

  // 공통 흐름은 mixin이 보유
  void onAnyChange() {                    // 입력 리스너가 호출
    setState(() => _saveStatus = AutoSaveStatus.pending);
    _debounceTimer?.cancel();
    _debounceTimer = Timer(_debounceDelay, _save);   // 0.6s debounce
    if (draftEnabled) {
      _draftDebounceTimer?.cancel();
      _draftDebounceTimer = Timer(_draftDebounceDelay, _persistDraftNow); // 0.3s
    }
  }
}

// 2) 화면은 with로 장착하고 빈칸만 채움
class _WriteJusojungSheetState extends ConsumerState<WriteJusojungSheet>
    with AutoSaveControllerMixin<WriteJusojungSheet, Jusojung>
    implements WriteSheetFlushable {
  @override
  void initState() {
    super.initState();
    _controller.addListener(onAnyChange);  // ← 타이핑 → mixin 흐름 진입
    maybeRestoreDraft();                    // ← 강제종료 후 복원
  }
  @override Jusojung buildRecord() => Jusojung(...);
  @override bool isContentEmpty() => _controller.text.trim().isEmpty;
  // performCreate/Update/Delete → ref.read(provider.notifier).xxx()
}
```

상태 흐름(헤더 인디케이터):
```text
idle ─타이핑→ pending ─0.6s→ saving ─성공→ saved ─3s→ idle
                                      └실패→ error ─재시도→ pending
```

닫기 3경로(모두 같은 flush 호출): 닫기버튼 / 뒤로가기(PopScope) / 바깥탭(barrier, GlobalKey로 State 접근).

## 흔한 함정 / 헷갈리는 점
- **`await` 이후 `ref` 접근 금지.** `flushPendingSave()`는 await 동안 시트가 dispose될 수 있어, await **이전**(확실히 mounted)에 `storage`/`key`를 지역변수로 캡처해 둔다. 안 그러면 "Cannot use ref after the widget was disposed". → [[2026-06-05-riverpod-ref-context-async-gap]]
- **`_inFlightSave`(진행 중 Future) 추적.** 저장 요청이 중복으로 두 번 날아가지 않게 하고, 닫을 때 "그 요청 끝날 때까지 기다려"를 가능하게 하는 핵심 변수. 없으면 race condition.
- **flush와 lifecycle flush의 차이는 "드래프트 삭제 여부".** 정상 close(`flushPendingSave`)=저장 후 **삭제**. 백그라운드 전환(`_flushApi`)=저장만 하고 **삭제 안 함**(안전망 유지).
- **mixin은 `WidgetsBindingObserver`를 직접 implements 못 함**(mixin-on-State 다이아몬드). → 별도 `_LifecycleHook` 클래스를 만들어 위임.
- **dispose에서 모든 Timer cancel + removeObserver** 필수. 안 하면 죽은 위젯에 setState → 예외/누수. → [[2026-06-06-flutter-dispose]]
- **debounce delay 리셋을 빼먹으면** debounce가 아니라 그냥 "첫 입력 후 N초 뒤 1회" 동작이 된다. 매번 `cancel()` 후 재생성이 본질.
- **읽는 순서:** 위→아래로 읽지 말 것. `enum AutoSaveStatus` → `onAnyChange()` → `_save()/_runSave()` → `flushPendingSave()` → abstract 목록과 구현체를 좌우로 대조.

## 질문 기록
### 내가 던진 질문 (원문)
- "lib/features/consultation/presentation/widgets/write_sheet/auto_save_controller.dart 이 파일 프로젝트를 처음 시작하느 낸가 이해하기 너무 어려운데 보다 자세히 해석하는 법이랑 어떻게 연동 되고 흘러가는지 플로우 다이어그램? 같은 것들도 잇었으면 좋겠는데"

### Claude가 되물은 확인 질문 + 답
- Q: 이 설명을 docs에 마크다운으로 저장할지, 아니면 특정 부분(_inFlightSave race condition, Mixin 다이아몬드 문제)을 더 깊이 파고들지? / A: `/dev-log`로 학습 노트화하는 것으로 갈음(이 노트가 그 산출물).

## 관련
- 발단이 된 이슈: (버그 아님 — 코드 해석/학습 세션)
- 관련 노트: [[2026-06-06-flutter-dispose]], [[2026-06-05-riverpod-ref-context-async-gap]], [[2026-06-06-flutter-진료실-데이터흐름-구조]]
- 코드: `lib/features/consultation/presentation/widgets/write_sheet/auto_save_controller.dart`, `lib/core/storage/write_draft_storage.dart`

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
