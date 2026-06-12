# Flutter 바텀시트 "닫기 3경로(X·딤드·스와이프)" 저장 일원화 — PopupRoute beforeClose + WriteSheetFlushable

> 한 줄 요약: 시트를 닫는 길은 X버튼만이 아니라 딤드 탭·아래 스와이프(·시스템 뒤로)까지 여럿이다. 저장을 어느 한 경로에 묶지 말고, "닫히기 직전 훅(`beforeClose`)" 으로 **모든 경로를 단일 flush 메서드에 수렴**시킨다.

## 핵심
- `showModalBottomSheet` 대신 커스텀 `PopupRoute`(여기선 `WriteSheetRoute`)를 쓰면 닫기 트리거가 여러 개 생긴다:
  1. 시트 안의 **X(닫기) 버튼** — 내가 짠 핸들러가 pop.
  2. **딤드(배리어) 영역 탭** — `barrierDismissible == true` 일 때 라우트가 **자동으로** pop.
  3. **아래로 스와이프(drag-to-dismiss)** — 시트 컨테이너가 드래그 임계 넘으면 pop.
- 함정: 1번 핸들러에만 "저장 + 토스트"를 넣으면 2·3번은 **개발자 코드를 안 거치고** pop 되어 변경분이 조용히 사라진다.
- 해법(2가지를 같이 적용):
  - **flush 일원화** — 저장/토스트 로직을 `flushPendingSave()` 한 메서드(SSOT)로 모으고, `bool _flushed` 가드로 닫기당 1회 보장.
  - **beforeClose 훅** — 라우트가 어떤 경로로 닫히든 pop **직전에** `beforeClose()` 를 호출하게 하고, 그 안에서 `flushPendingSave()` 를 부른다.
- 저장값을 **`Navigator.pop(result)` 의 반환값에 의존하지 않는다.** dismiss 경로는 `pop()` 을 인자 없이 부르므로 result 가 null → 유실. 대신 `onChanged` 콜백으로 부모에 직접 전달.

## 언제 / 왜 쓰나
- 작성/편집 바텀시트(상병·주소증·메모·오더 등)에서 "닫으면 저장"인 화면.
- 닫기 수단이 둘 이상(특히 딤드 dismiss, 드래그 dismiss 허용)일 때 필수.
- "X로는 저장되는데 배경 탭/스와이프로 닫으면 안 된다"는 버그를 구조적으로 차단.

## 예시 (코드·명령)

### (1) 라우트 — 닫히기 직전 훅 제공
```dart
class WriteSheetRoute<T> extends PopupRoute<T> {
  WriteSheetRoute({required this.child, this.beforeClose});
  final Widget child;
  final Future<void> Function()? beforeClose;   // 닫히기 전 flush 훅

  @override
  bool get barrierDismissible => true;          // 딤드 탭으로 닫기 허용

  @override
  Widget buildPage(context, animation, secondaryAnimation) => BottomSheetContainer(
        animation: animation,
        onBarrierTap: () async {                // 딤드 탭 경로
          await beforeClose?.call();            //   → flush 먼저
          if (context.mounted) Navigator.of(context).pop();
        },
        beforeClose: beforeClose,               // 스와이프 dismiss 경로도 같은 훅
        child: child,
      );
}
```

### (2) 시트 컨테이너 — 스와이프 dismiss 도 같은 훅을 거침
```dart
void _onDragEnd(DragEndDetails details) async {
  final dismiss = _dragOffset > sheetHeight * _dismissThreshold;
  if (dismiss) {
    await widget.beforeClose?.call();           // 스와이프 경로 → flush
    if (mounted) Navigator.of(context).pop();
  } else {
    setState(() => _dragOffset = 0);
  }
}
```

### (3) 시트 State — flush 일원화 (저장+토스트 SSOT)
```dart
abstract class WriteSheetFlushable {            // 공용 인터페이스
  Future<void> flushPendingSave();
}

class _MySheetState extends State<MySheet> implements WriteSheetFlushable {
  bool _flushed = false;

  @override
  Future<void> flushPendingSave() async {
    if (_flushed) return;                       // 닫기당 1회 (이중호출 가드)
    _flushed = true;
    if (!mounted) return;
    if (_isChanged) {                           // 변경 없으면 저장/토스트 skip
      showToast(context, '저장되었습니다.');
      widget.onChanged?.call(_value);
    }
  }

  Future<void> _close() async {                 // X버튼도 같은 경로로 통일
    await flushPendingSave();
    if (mounted) Navigator.of(context).pop();
  }
}
```

### (4) 호출부 — GlobalKey 로 시트 State 를 잡아 beforeClose 에 연결
```dart
final sheetKey = GlobalKey<State<MySheet>>();
Navigator.of(context).push<void>(
  WriteSheetRoute(
    beforeClose: () async {
      final s = sheetKey.currentState;
      if (s is WriteSheetFlushable) await (s as WriteSheetFlushable).flushPendingSave();
    },
    child: MySheet(key: sheetKey, onChanged: onValueChanged),  // 저장은 콜백 경로
  ),
);
```

---

### 👉 "이 패턴을 손으로 만들어 보며 이해"하기 위한 예제 생성 프롬프트
> 아래 블록을 그대로 새 Claude 세션(또는 DartPad용)에 붙여넣으면, 위 패턴을 **최소 실행 예제**로 재현해 준다.
> 실제 프로젝트 코드(테마/토큰/Riverpod)를 걷어내고 **순수 Flutter 표준 위젯만**으로 본질만 남기는 게 목적.

```text
역할: 너는 Flutter 교사다. 아래 "닫기 경로 통일 + flush 일원화" 패턴을 내가 직접 돌려보며
이해할 수 있는 단일 파일 최소 예제(main.dart 하나, 외부 패키지 0개, DartPad에서 실행 가능)를 만들어줘.

재현할 패턴:
- 커스텀 PopupRoute 로 띄우는 바텀시트. 닫는 방법이 3가지다:
  (a) 시트 안 "닫기" 버튼, (b) 딤드(배리어) 영역 탭, (c) 시트를 아래로 드래그해서 dismiss.
- 시트에는 TextField 하나가 있고, 값이 바뀌었을 때만(=_isChanged) 닫힐 때 SnackBar("저장되었습니다.")
  를 1번 띄우고 부모의 콜백으로 값을 올려, 부모 화면 텍스트를 갱신한다.
- 핵심: 저장+SnackBar 로직은 시트 State 의 flushPendingSave() 한 메서드에만 둔다(SSOT).
  _flushed bool 로 닫기당 1회만 실행되게 가드한다.
- 세 경로 모두 pop 직전에 beforeClose() 훅을 거쳐 flushPendingSave() 를 호출하도록 배선한다.
  저장값은 Navigator.pop(result) 의 반환값에 의존하지 말고 onChanged 콜백으로 전달한다.

요구사항:
1. 먼저 "왜 X버튼에만 저장을 넣으면 딤드/스와이프에서 값이 새는지" 를 3~4줄로 설명.
2. 그 다음 전체 main.dart (PopupRoute + 드래그 dismiss 컨테이너 + WriteSheetFlushable 인터페이스
   + flush 가드 + GlobalKey 배선) 를 주석과 함께 제시.
3. 마지막에 "이 예제로 직접 해볼 실험 3가지"(예: beforeClose 를 일부러 빼고 스와이프로 닫아 값이
   사라지는 것 관찰 / _flushed 가드를 제거해 토스트 2번 뜨는 것 관찰 / barrierDismissible=false 로 바꿔보기)를
   제안. 각 실험이 무엇을 증명하는지 한 줄씩.

제약: Material 위젯만, 상태관리 패키지 없이 StatefulWidget+setState, 한 파일.
```

## 흔한 함정 / 헷갈리는 점
- **`barrierDismissible: true` 면 딤드 dismiss 가 자동**이라 내 핸들러를 안 거친다. → flush 누락의 1순위 원인.
- **드래그 dismiss 는 라우트가 아니라 "시트 컨테이너"가 처리**하는 경우가 많다(`onVerticalDragEnd`). 라우트에만 `beforeClose` 를 꽂고 컨테이너엔 안 꽂으면 스와이프 경로만 샌다 → 두 곳 다 연결해야 함.
- **`pop(result)` 반환값 의존**: dismiss 경로는 `pop()` 인자 없이 → `result == null`. 저장을 콜백으로 옮겨야 견고.
- **이중 호출**: 딤드 탭 후 위젯 dispose 등으로 flush 가 두 번 불릴 수 있다 → `_flushed` 가드로 1회 보장(토스트 중복 방지).
- **`mounted` 가드**: flush 안에서 `context` 사용(토스트) 전 `if (!mounted) return;` 필수(async gap).
- 자동저장(debounce)까지 있는 시트라면, `flushPendingSave` 가 **in-flight 저장을 await** 해서 닫힘과 경합하지 않게 해야 함 → 자매 노트의 mixin 패턴 참고.

## 질문 기록
### 내가 던진 질문 (원문)
- "딤드·스와이프 후속도 있어?"
- "로 내가 해당 코드를 이해하기 위해 예 코드를 만드는 것에 대한 프롬프트를 작성하도록 도와줄래? 그리고 현재 코드도 정리하고"

### Claude가 되물은 확인 질문 + 답
- Q: QA 문서 TC-27 을 ✅ 로 갱신해둘까요? / A: ❓ 미답

## 관련
- 발단이 된 이슈: [[2026-06-12-diagnosis-sheet-dim-swipe-save-missing]]
- 자매 노트(자동저장 mixin): [[2026-06-06-flutter-autosave-sheet-mixin-debounce]]
- 생명주기 가드: [[2026-06-05-riverpod-ref-context-async-gap]], [[2026-06-06-flutter-dispose]]

---
- 생성일: 2026-06-12
- 마지막 갱신: 2026-06-12
