# Flutter — 커서(캐럿) 위치 텍스트 삽입 & 포커스 보정

> 한 줄 요약: `TextEditingController.selection` 으로 캐럿 위치에 텍스트를 끼워넣고, `onTapOutside` 로 빠진 포커스는 postFrame 으로 되돌린다.

## 핵심
- `TextField` 의 입력 상태는 `TextEditingController` 가 들고 있고, 커서/선택영역은 `controller.selection`(`TextSelection`) 이다.
- **맨 끝 append** 와 **커서 위치 삽입** 은 다르다. 후자는 `selection.start/end` 로 본문을 잘라 그 사이에 넣는다.
- `controller.selection` 은 **포커스를 잃어도 offset 이 보존**된다 → 버튼을 눌러 포커스가 빠져도 직전 캐럿 위치에 삽입 가능.
- 포커스가 있으면 `EditableText` 가 selection 변경 시 캐럿을 자동으로 화면에 노출(`_showCaretOnScreen`)한다 → 보통 별도 스크롤 보정 불필요.

## 언제 / 왜 쓰나
- "지금 타이핑하던 자리에" 토큰/날짜/태그를 끼워넣는 인라인 삽입 UX.
- 반대로 "문단을 통째로 덧붙이는" 액션은 맨끝 append + `jumpTo(maxScrollExtent)` 가 더 맞다.

## 예시 (코드)
```dart
// 커서 위치 삽입 (selection 무효 시 맨끝 폴백, 앞 공백 보강)
void insertTextAtCursor(TextEditingController c, {required String text}) {
  final v = c.text;
  final s = c.selection;
  if (!s.isValid) {                      // baseOffset/extentOffset < 0 (포커스 없음/유실)
    c.text = v + text;
    c.selection = TextSelection.collapsed(offset: c.text.length);
    return;
  }
  final len = v.length;
  final start = s.start.clamp(0, len);   // stale selection 방어
  final end = s.end.clamp(0, len);
  final before = v.substring(0, start), after = v.substring(end);
  final needSpace = before.isNotEmpty && !before.endsWith(' ') && !before.endsWith('\n');
  final insert = (needSpace ? ' ' : '') + text;
  c.value = TextEditingValue(
    text: before + insert + after,
    selection: TextSelection.collapsed(offset: before.length + insert.length),
  );
}
```

```dart
// 버튼 탭 → onTapOutside 가 unfocus 를 먼저 태운다. 재포커스는 postFrame 으로 미뤄 순서 보장.
void onInsertButton() {
  insertTextAtCursor(controller, text: '...');
  WidgetsBinding.instance.addPostFrameCallback((_) {
    if (mounted) focusNode.requestFocus();
  });
}
```

## 흔한 함정 / 헷갈리는 점
- **`selection` 이 -1 인 케이스**: 한 번도 포커스 안 준 필드. 그대로 substring 하면 깨진다 → `isValid` 체크 + 맨끝 폴백 필수. (맨 앞 폴백은 데이터 훼손)
- **stale offset**: 텍스트가 줄어든 뒤 옛 selection 이 길이를 초과할 수 있다 → `clamp(0, length)`.
- **`controller.text = ...` 직접 대입 시 selection 리셋**: 텍스트만 바꾸면 커서가 0으로 튄다 → `controller.value = TextEditingValue(text:, selection:)` 로 **함께** 세팅.
- **같은 탭 안의 unfocus vs requestFocus 경쟁**: 버튼의 `onTapOutside`(PointerDown) 가 `onTap` 보다 먼저 unfocus 를 태운다. 동기 requestFocus 는 묻힐 수 있어 `addPostFrameCallback` 으로 미룬다.

## 관련
- 발단이 된 작업: [[2026-06-06-write-sheet-text-insertion]]
- 참고: `TextEditingController`, `TextSelection`, `EditableText` (Flutter SDK)

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
