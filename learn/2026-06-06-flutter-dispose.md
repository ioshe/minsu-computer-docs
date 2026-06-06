# Flutter `dispose()` — 위젯 생명주기 정리 메서드

> 한 줄 요약: `StatefulWidget`의 `State`가 화면에서 완전히 제거될 때 딱 한 번 호출되며, **내가 직접 만든 컨트롤러·노드·구독·리스너를 반납**하는 자리다. 안 치우면 메모리 누수가 난다.

## 핵심
- `dispose()`는 `State` 생명주기의 **마지막** 단계다. (`createState` → `initState` → `build` … → `dispose`)
- GC가 자동으로 못 거두는, **계속 살아있으면서 메모리/리스너를 붙잡는 객체**를 수동 해제하는 용도다.
- 원칙은 **"생성↔해제 페어"**: `initState`(또는 필드 초기화)에서 내가 `new` 한 것은 `dispose`에서 내가 치운다. 내가 안 만든 것(부모/Provider 소유)은 내가 치우지 않는다.
- `StatelessWidget`에는 `dispose`가 아예 없다.

## 언제 / 왜 쓰나
GC가 자동 회수하지 못하는 리소스를 쥐고 있을 때:

| 종류 | 예시 |
|------|------|
| 컨트롤러 | `TextEditingController`, `ScrollController`, `AnimationController` |
| 포커스 | `FocusNode` |
| 리스너 등록 | `addListener` 한 것 → `removeListener` |
| 구독/타이머 | `StreamSubscription`, `Timer`, `Ticker` |

이런 객체는 위젯이 사라져도 내부 참조/리스너가 남아 **메모리 누수**를 일으키므로 명시적으로 해제해야 한다.

## 예시 (코드)
실제 프로젝트 코드 — `lib/features/consultation/presentation/widgets/etc/write_etc_sheet.dart`:

```dart
@override
void dispose() {
  _controller.removeListener(onAnyChange);  // ① 등록한 리스너부터 해제
  _controller.dispose();                    // ② 컨트롤러 해제
  _textScrollController.dispose();          // ③ 스크롤 컨트롤러 해제
  _textFocusNode.dispose();                 // ④ 포커스노드 해제
  super.dispose();                          // ⑤ 항상 맨 마지막
}
```

이 위젯은 `_controller`, `_textScrollController`, `_textFocusNode`를 직접 생성했으므로 직접 반납한다.

## 흔한 함정 / 헷갈리는 점
- **`super.dispose()`는 반드시 맨 마지막 줄.** 먼저 호출하면 이후 정리 로직이 깨질 수 있다.
- **`addListener` 했으면 `removeListener` 먼저** 한 뒤 `dispose()`. 순서 뒤집히거나 빠지면 누수.
- **dispose 이후 `setState` 금지.** 비동기 콜백(`await` 후)에서 위젯이 이미 사라졌을 수 있으므로 `if (mounted)` 가드를 둔다.
- **내가 안 만든 컨트롤러는 dispose 하지 마라.** Riverpod provider나 부모가 소유한 객체를 자식이 dispose하면 다른 곳에서 죽은 객체를 참조하게 된다.
- 생성↔해제 짝이 안 맞는 것(생성은 했는데 dispose 누락, 또는 그 반대)이 가장 흔한 버그.

## 질문 기록
### 내가 던진 질문 (원문)
- "flutter 에서 dispose 가 어떤때 스이는겨"

### Claude가 되물은 확인 질문 + 답
- (없음 — 개념 질문이라 추가 확인 불필요)

## 관련
- 발단이 된 이슈: (없음 — 사건이 아닌 개념 학습)
- 참고: `State.dispose` API 문서 (Flutter docs)

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
