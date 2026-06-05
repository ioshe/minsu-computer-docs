# 2026-06-05 · 주소증 mock→API 전환 후 "Cannot use ref after the widget was disposed" 크래시

## 1. 개요
- 날짜: 2026-06-05
- 프로젝트 / 화면 / 모듈: devops-mediclo / 진료실 주소증(jusojung) 작성·수정 바텀시트 / consultation feature
- 이슈 유형: 크래시 (런타임 예외)
- 계층: 클라이언트 (Flutter / Riverpod 상태관리 · 위젯 생명주기)
- 최종 상태: 해결 (코드 수정 + `flutter analyze` 통과 / 런타임 재확인은 사용자 진행 예정)
- 중요도: 보통

## 2. 환경
- OS / 기기 / 웹뷰: Linux (개발 머신) / 태블릿 타깃 (Flutter)
- 앱 버전 / 빌드: 개발 빌드 (debug)
- 브랜치 / 커밋: `develop`
- 관련 의존성·버전: flutter_riverpod 2.6.1, riverpod_annotation 2.6.1, riverpod_generator 2.6.4

## 3. 증상
### 관찰된 현상
- 주소증 mock 데이터를 실제 API(`patient-memo`) 서비스 계층으로 전환한 직후, 주소증 작성/수정 시트를 닫는 흐름에서 런타임 예외 발생.
- 전환 이전(provider가 in-memory mock 직접 사용)에는 재현되지 않던 회귀(regression).

### 재현 절차
1. 진료실 → 주소증 탭에서 작성/수정 시트 열기
2. 내용 입력 (auto-save 가 debounce 후 저장 시작)
3. 저장(create/update)이 진행 중인 상태에서 시트를 닫음(barrier 탭 / X)
4. 닫히는 도중 예외 발생

### 에러 메시지 / 스택 트레이스 (원문)
```text
Another exception was thrown: Bad state: Cannot use "ref" after the widget was disposed.
```
> 스택 트레이스 전문은 캡처하지 않음(한 줄 메시지만 확보). 코드 정독으로 발생 지점을 특정함.

## 4. 확인한 내용
- 호출 경로 추적: 시트 close = `write_sheet_route.dart`의 `onBarrierTap` → `await beforeClose()`(= `AutoSaveControllerMixin.flushPendingSave()`) → `if (context.mounted) Navigator.pop()`.
- `flushPendingSave()` 내부:
  - `await _inFlightSave` 또는 `await _save()` 로 진행 중 저장 완료를 대기
  - 그 **await 뒤** `if (draftEnabled) await draftStorage.remove(draftKey);` 실행
  - `draftStorage` getter = `ref.read(writeDraftStorageProvider)` (위젯 ConsumerState의 WidgetRef)
- 결과 해석: 저장을 in-memory(즉시) → 실제 async(150ms~네트워크)로 바꾸면서 **await 시간이 길어졌고**, 그 사이 시트가 dispose되면 마지막 `draftStorage` 접근이 **dispose된 위젯의 ref**를 건드려 예외 발생.

## 5. 원인 분석
- **확정 원인:** `AutoSaveControllerMixin.flushPendingSave()`가 비동기 저장을 `await` 한 **뒤** `draftStorage`(= `ref.read(...)`)에 접근. 저장이 실제 async로 느려지자 await 동안 시트 dispose가 끼어들 윈도우가 생겨, async gap을 넘긴 WidgetRef 접근이 터짐.
- **유발 트리거(회귀 원인):** 주소증 provider(`CurrentJusojungList`)의 create/update/delete를 in-memory에서 `await _service.xxx()`(실서버/모킹 지연 포함)로 교체 → 저장 시간이 길어져 잠재 버그가 표면화.
- **동일 패턴 잠재 지점:** `jusojung_card.dart` 삭제 핸들러가 `await showAppDialog(...)` **뒤** `ref.read(...notifier)` 사용 (이번 에러와는 별개지만 같은 종류의 잠재 크래시).

## 6. 조치 내역
- **근본 수정 (root fix)** — `lib/.../write_sheet/auto_save_controller.dart` `flushPendingSave()`:
  - `ref.read` 기반 `draftStorage`와 `draftKey`를 **await 이전(확실히 mounted)** 에 지역변수로 캡처, await 뒤에는 캡처본만 사용.
  ```dart
  Future<void> flushPendingSave() async {
    final storage = draftEnabled ? draftStorage : null; // await 전 ref 캡처
    final key = draftKey;
    _debounceTimer?.cancel();
    _draftDebounceTimer?.cancel();
    if (_inFlightSave != null) {
      await _inFlightSave;
    } else if (_saveStatus == AutoSaveStatus.pending) {
      await _save();
    }
    if (storage != null) {
      await storage.remove(key); // ref 재접근 없음
    }
  }
  ```
  - 이 mixin은 memo/기타 시트도 공유하므로 동일 크래시가 세 곳 모두 예방됨.
- **예방 수정** — `lib/.../widgets/jusojung/jusojung_card.dart` `_onDeleteTap()`:
  - `notifier`/`id`를 `showAppDialog` await **이전**에 캡처하도록 이동.
  ```dart
  final notifier = ref.read(currentJusojungListProvider(widget.scheduleId).notifier);
  final id = widget.entry.id;
  final confirmed = await showAppDialog<bool>(...);
  if (confirmed != true) return;
  notifier.remove(id);
  ```
- 조치 결과: 두 파일 모두 `flutter analyze` → **No issues found**.

## 7. 최종 상태
- 해결 여부: 코드 수정 완료, 정적 분석 통과. 런타임 재현 테스트는 사용자가 hot restart 후 확인 예정.
- 남은 문제 / 추가 확인 필요:
  - 디버깅용 `debugPrint('[MyDebug] ...')` (current_jusojung_provider.dart createJusojung) 정리 필요
  - `_customerId`가 아직 하드코딩(500497) — 실제 환자 컨텍스트 provider 연결 남음(별개 작업)

## 8. 학습 포인트
- 핵심 개념: **Riverpod `WidgetRef`(및 `BuildContext`)는 `await`(async gap)를 넘겨 살아남는다는 보장이 없다.** dispose 이후 접근하면 예외.
- 이번에 배운 점: "동기일 땐 안 터지던" 코드가 **비동기로 느려지는 순간** 잠재 dispose 레이스가 표면화된다. mock→실API 전환은 이런 회귀를 부르는 전형적 변경.
- 다음에 같은 증상이면 확인할 순서: 1) 메시지의 `ref`/`context`가 누구 것인지(Widget vs Notifier) 2) 그 사용처가 `await` **뒤**인지 3) 그 await가 최근에 더 느려졌는지(동기→비동기)
- 👉 별도 학습 노트: [[2026-06-05-riverpod-ref-context-async-gap]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 비동기 핸들러에서 `await` 뒤에 `ref`/`context`를 쓰지 않는다 (필요한 객체는 await 전에 캡처)
  - [ ] mock→실API 등 "동기→비동기" 전환 시, 닫기/취소 등 생명주기 종료 경로를 재점검
  - [ ] `context` 사용은 `if (context.mounted)` 가드
- 추가하면 좋을 테스트 / 가드 / 로깅: 시트 저장 중 즉시 닫기 위젯 테스트(저장 in-flight 중 dispose) 추가
- 자동화 후보: lint `use_build_context_synchronously` 는 context엔 걸리지만 ref엔 안 걸림 → ref용 커스텀 lint/리뷰 체크리스트로 보완

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `Bad state: Cannot use "ref" after the widget was disposed` (또는 `...context after disposed`)가 비동기 작업 완료 직후 발생
- 확인 절차: 해당 화면의 async 핸들러에서 `await` 다음 줄들에 `ref.`/`ref.read`/getter(내부가 ref.read)/`context` 사용이 있는지 grep
- 조치 절차: await 이전에 `final x = ref.read(...)` / `final c = context...`로 캡처 → await 뒤엔 캡처본만 사용. context 출력류는 `if (mounted)` 가드.
- 정상 확인 기준: 저장 in-flight 중 시트를 닫아도 예외 없이 닫히고, 드래프트/상태가 일관되게 정리됨

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "Another exception was thrown: Bad state: Cannot use "ref" after the widget was disposed."  (※ 질문이라기보다 에러 제보로 시작)

### Claude가 되물은 확인 질문 + 답
- Q: (jusojung_card 삭제 핸들러의 동일 잠재 패턴도) "원하면 같이 고쳐드릴게요." → 같이 고칠까요?
  - A: "ㅇㅇ" (예 — 함께 수정 진행)

## 12. 한 줄 결론
> 동기였던 저장을 실제 async로 바꾸자 숨어 있던 dispose 레이스가 드러난 회귀 — **`ref`/`context`는 await 전에 캡처**해서 async gap을 넘기지 않는 게 정답.
