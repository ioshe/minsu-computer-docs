# Riverpod `ref` / `context` 는 async gap(await)을 넘기지 마라

> 한 줄 요약: 위젯의 `WidgetRef`/`BuildContext`는 `await` 동안 위젯이 dispose되면 무효가 된다. 비동기 전에 필요한 객체를 **캡처**하고, await 뒤에는 캡처본만 쓴다.

## 핵심
- Flutter 위젯(특히 `ConsumerState`)의 `ref`(WidgetRef)와 `context`는 **그 위젯이 살아있는 동안만 유효**하다.
- `await` 한 줄은 "여기서 실행이 멈췄다가 나중에 재개된다"는 뜻 = **async gap**. 그 사이 사용자가 화면을 닫거나 위젯이 리빌드로 사라지면(dispose) 위젯은 죽는다.
- 죽은 뒤 `ref`에 접근하면: `Bad state: Cannot use "ref" after the widget was disposed`.
  `context`에 접근하면: deactivated widget 관련 에러 / 잘못된 동작.
- 무서운 점: **동기 코드일 땐 gap이 거의 0이라 안 터지다가**, 그 부분이 비동기(네트워크·디스크·지연)로 바뀌면 gap이 커져 잠재 버그가 표면화된다. (mock→실API 전환이 대표적)
- 주의: getter가 내부에서 `ref.read(...)`를 호출하면 그 getter도 "ref 접근"이다. 호출 위치가 await 뒤면 동일하게 터진다.

## 언제 / 왜 쓰나
- 비동기 이벤트 핸들러(저장 후 정리, 다이얼로그 확인 후 처리, 닫기 전 flush 등) 작성 시 항상.
- "mock/동기 → 실제 async"로 바꾸는 리팩토링 시, 생명주기 종료 경로(닫기/취소/뒤로가기)를 반드시 재점검.

## 예시 (코드·명령)
```dart
// ❌ 나쁨 — await 뒤에 ref/context 접근
Future<void> onDelete() async {
  final ok = await showDialog<bool>(...);      // 이 사이 위젯이 dispose될 수 있음
  if (ok != true) return;
  ref.read(listProvider.notifier).remove(id);  // 💥 ref after disposed
}

// ✅ 좋음 — await 이전에 캡처
Future<void> onDelete() async {
  final notifier = ref.read(listProvider.notifier); // gap 전에 캡처
  final id = entry.id;
  final ok = await showDialog<bool>(...);
  if (ok != true) return;
  notifier.remove(id);                              // 캡처본만 사용
  if (mounted) showToast(context, '삭제됨');         // context 는 mounted 가드
}

// ✅ getter(내부 ref.read)도 동일하게 캡처
Future<void> flush() async {
  final storage = draftStorage; // == ref.read(...) → await 전에 캡처
  final key = draftKey;
  await _inFlightSave;          // 느린 비동기
  await storage.remove(key);    // ref 재접근 없음
}
```

## 흔한 함정 / 헷갈리는 점
- **"누구의" ref/context인지 구분**: 에러 메시지 "after the **widget** was disposed"는 위젯(`ConsumerState`)의 WidgetRef. Notifier/Provider 내부 `Ref`는 "after it was disposed"로 메시지가 다르다. 잡는 곳이 다르다.
- lint `use_build_context_synchronously`는 **context**엔 경고를 주지만 **ref**엔 안 준다 → ref는 사람이 챙겨야 한다.
- `if (mounted)` 가드는 "건너뛰기"라 부수효과(드래프트 삭제 등)가 스킵될 수 있다. 그 작업을 꼭 끝내야 하면 **가드보다 캡처**가 낫다(캡처한 객체는 위젯과 무관하게 동작).
- Notifier 내부의 `ref`(예: `ref.read(serviceProvider)`)는 위젯 ref가 아니라 provider ref라, 위젯 dispose와 무관. 혼동 주의.

## 질문 기록
### 내가 던진 질문 (원문)
- (이번 세션에 이 학습 노트 주제로 사용자가 직접 던진 질문 없음 — 이슈에서 파생)

### Claude가 되물은 확인 질문 + 답
- (해당 없음)

## 관련
- 발단이 된 이슈: [[2026-06-05-riverpod-ref-after-widget-disposed]]
- 참고 링크: Flutter `BuildContext` across async gaps (use_build_context_synchronously lint)

---
- 생성일: 2026-06-05
- 마지막 갱신: 2026-06-05
