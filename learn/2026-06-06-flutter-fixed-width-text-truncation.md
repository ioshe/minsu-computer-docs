# Flutter 고정폭 위젯에서 텍스트가 잘리는 두 원인 (한글 softWrap + 시스템 textScaler)

> 한 줄 요약: 크기가 고정된 배지/칩에서 한글 라벨 끝 글자가 잘리면, ① 한글 음절 줄바꿈(`softWrap`)과 ② 시스템 글자배율(`textScaler`)을 의심하라. 그리고 폭을 잴 땐 `textScaler`를 꼭 포함해라.

## 핵심
- **① 한글 음절 softWrap**: `Text`의 `softWrap`은 기본 true다. Flutter는 한글을 **음절 단위로 줄바꿈 가능**으로 보기 때문에, `maxLines:1`이고 폭이 1~2px만 빠듯하면 `아웃콜`을 `아웃`/`콜`로 쪼갠 뒤 첫 줄만 남겨 `콜`이 사라진다. → 단일 라인 강제 `softWrap:false`.
- **② 시스템 textScaler**: 기기의 "글자 크게"(접근성) 설정은 앱의 모든 `Text`에 곱해진다(예 1.10). **크기가 고정된 배지/칩은 물리적으로 못 커져서** 글자가 박스를 넘쳐 잘린다. 전역에서 textScaler를 고정(`MediaQuery` override)하지 않은 앱이면 기기 설정이 그대로 먹는다. → 고정크기 칩은 `textScaler: TextScaler.noScaling`.
- **함정**: `TextPainter`로 "들어가나?" 측정할 때 `MediaQuery.textScalerOf(context)`를 안 넘기면 실제보다 좁게(예 38.44) 나와 `fits=true`로 **오판**한다. 실제(42.33)는 textScaler를 곱한 값이다.
- **또 다른 함정**: DevTools에서 `Text`의 `size.width`는 **부모 박스에 맞춰 잘린 값**으로 보고된다. 그래서 박스 안쪽폭과 "딱 같게(40=40)" 보여 "맞네?"로 속는다. 글자가 원하는 진짜 폭은 더 큰데 안 보인다.
- **경고가 없다**: 노란-검정 OVERFLOW 줄무늬는 `Row/Column`(Flex) 넘침에만 뜬다. **Text 잘림은 경고가 안 뜨고 조용히 clip**된다.

## 언제 / 왜 쓰나
- 고정 크기 배지·칩·태그·버튼 라벨이 특정 단어(특히 한글 3자+)에서만 끝 글자가 잘릴 때.
- "디자인 토큰은 Figma와 똑같은데 실기기에서만 잘린다"면 십중팔구 textScaler.

## 예시 (코드·명령)
```dart
// 고정폭 배지 라벨 — 두 처방 같이
Text(
  label,
  maxLines: 1,
  softWrap: false,                  // ① 한글 음절 줄바꿈 방지
  overflow: TextOverflow.clip,
  textScaler: TextScaler.noScaling, // ② 시스템 글자배율 무시(고정크기)
)

// "이 글자가 이 박스에 들어가나?" 정확히 재기 (실앱에 심기)
assert(() {
  final ts = MediaQuery.textScalerOf(context);   // ← 반드시 포함!
  final tp = TextPainter(
    text: TextSpan(text: label, style: realStyle),
    textDirection: TextDirection.ltr,
    maxLines: 1,
    textScaler: ts,
  )..layout();
  debugPrint('textW=${tp.width} vs innerWidth');
  return true;  // assert(() {...}()) → 릴리즈에서 자동 제거(트리쉐이킹)
}());
```

디버그 코드 넣는 위치:
- 전역 `debugPaintSizeEnabled = true` → `main()` 맨 위(또는 DevTools "Debug Paint" 버튼, 코드 없이).
- 특정 위젯 값 측정 → 그 위젯 `build()` 안, `assert(() {...}())`로 감싸기.

## 흔한 함정 / 헷갈리는 점
- `fontSize`(예 15.0)와 `width`(예 37.5)를 혼동하지 말 것. 잘림 판단은 **width**만 본다.
- 앱 자체 화면배율(여기선 `TokenScale.scaledTextStyle`, 화면 긴 축 기준 ×1.0/1.25/1.5)과 **시스템 textScaler는 별개**다. 둘 다 곱해진다.
- flutter_test/headless 렌더는 textScaler=1.0이라 실기기와 다르다 → ground truth는 **실앱 계측**.
- `softWrap:false`만으론 부족할 수 있다(넘치면 clip만 됨). 고정크기면 `textScaler.noScaling`까지 같이.

## 질문 기록
### 내가 던진 질문 (원문)
- "자간 -1% 이거때문에 그런거 아니야?"
- "플러터 처음 하느 ㄴ사람도 이해할 수 있게 말해줄래? 아웃콜에서 콜 짜가 계속 짤려보이기는하는데,,,"
- "잘보인다 근데 flutter debug 에서 도 ㅎ알 수 있나..?"
- "devtools를 열었는데 Text 에서는 어느 값을 봐야 하는지..? ... 그 위 container 에서는 어떤 값을 봐야하느지"
- "그리고 디버깅 코드들은 넣는다면 어디에 넣는거니"

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 기록할 질문 없음 — 학습 노트 한정)

## 관련
- 발단이 된 이슈: [[2026-06-06-memo-badge-text-truncation]]
- 참고: Flutter `TextScaler`, `MediaQuery.textScalerOf`, `debugPaintSizeEnabled`

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
