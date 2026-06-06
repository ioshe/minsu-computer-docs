# 2026-06-06 · 메모 유형 배지에서 '아웃콜'의 '콜'이 잘리는 문제

## 1. 개요
- 날짜: 2026-06-06
- 프로젝트 / 화면 / 모듈: devops-mediclo / 메모 카드 / `MemoTypeBadge` (유형 배지 40×24)
- 이슈 유형: 버그 (UI 텍스트 렌더링 잘림)
- 계층: 클라이언트 (UI/레이아웃)
- 최종 상태: 해결
- 중요도: 보통

## 2. 환경
- OS / 기기 / 웹뷰: Android 태블릿 (실기기, `I/flutter` 로그), **시스템 글자 크기 110%로 설정됨**
- 앱 버전 / 빌드: debug 빌드
- 브랜치 / 커밋: develop
- 관련 파일:
  - `lib/features/consultation/presentation/widgets/memo/atoms/memo_type_badge.dart`
  - 토큰: `lib/config/design_tokens/token_scale.dart` (`scaledTextStyle`), `typography.dart` (`caption12SB`)

## 3. 증상
### 관찰된 현상
- 메모 카드의 유형 배지에서 라벨 `아웃콜`의 마지막 글자 `콜`이 사라지거나(=`아웃`만 보임), 수정 도중엔 `콜`이 반만 보임.
- 처음엔 "배지 글자 크기가 안 맞는다 / 가운데 정렬이 안 맞는다"로 보였음 (실제론 잘림 때문에 어색했던 것).
- `메모1`/`메모2`/`인콜`은 정상. `아웃콜`(한글 3자)만 문제.

### 재현 절차
1. 시스템(태블릿) 글자 크기를 110%로 설정.
2. 메모 카드 목록에서 유형이 `아웃콜`인 메모를 표시.
3. 배지의 `콜`이 잘림.

### 로그 (원문, 디버그 계측으로 수집)
```text
# 1차 계측 (textScaler 미반영 — 오판)
[BADGE-DBG] label=아웃콜 scale=1.250 fontSize=15.00 box=50.00 inner=40.00 textW=38.44 fits=true

# 2차 계측 (시스템 textScaler 반영 — 진실)
[BADGE-DBG2] label=메모2  textScaler=16.50/15 inner=40.00 realTextW=38.01 fits=true
[BADGE-DBG2] label=아웃콜 textScaler=16.50/15 inner=40.00 realTextW=42.33 fits=false
[BADGE-DBG2] label=인콜   textScaler=16.50/15 inner=40.00 realTextW=28.22 fits=true
[BADGE-DBG2] label=메모1  textScaler=16.50/15 inner=40.00 realTextW=35.64 fits=true
```

## 4. 확인한 내용
- **Figma 원본 대조** (file `wXpPSssWtkHiprKGf61kbQ`, ToggleButton 노드): 배지 = 고정 40px + 좌우 padding 4 + 텍스트 Pretendard SemiBold 12px(`Body-12SB`) + `overflow-hidden/ellipsis`. → 코드의 `caption12SB`(12px·w600·-0.12)와 폰트·박스·자간 전부 일치. 폰트 크기/자간은 원인 아님.
- **TextPainter 실측** (Pretendard 로드 후): @scale1.0 아웃콜 30.75px / @scale1.25 38.44px → 둘 다 박스 내부폭에 들어감. "폭 초과 아님"으로 1차 오판.
- **실위젯 PNG 렌더**(RepaintBoundary.toImage): softWrap 기본값에서 `아웃콜`이 `아웃`으로 잘림을 눈으로 확인. `softWrap:false`로 바꾸면 한 줄로 `아웃콜` 표시됨을 확인.
- **실앱 assert 계측**: `scale=1.25`(TokenScale), `textScaler=1.10`(시스템). 1차 계측이 `textScaler`를 빼서 38.44(fits=true)로 오판 → 2차에서 `MediaQuery.textScalerOf(context)` 반영하니 42.33 > 40 (fits=false)로 진실 확인.

## 5. 원인 분석
- **확정 원인 (두 가지가 겹침):**
  1. **한글 음절 softWrap** — `maxLines:1`인데 `softWrap`이 기본 true라, Flutter가 한글을 음절 단위 줄바꿈으로 처리해 `아웃`/`콜`로 쪼갠 뒤 첫 줄만 남겨 `콜`이 사라짐. 폭이 1~2px만 빠듯해도 발생.
  2. **시스템 textScaler(접근성 큰 글씨) 1.10** — 고정폭 40px 배지는 못 커지는데 글자만 ×1.1 되어 `아웃콜`이 42.33px로 박스(내부 40px)를 넘쳐 `콜`이 clip됨. 이 앱은 전역에서 textScaler를 고정하지 않아(main에 MediaQuery override 없음) 기기 설정이 그대로 먹힘.
- 1번을 먼저 잡으면 줄바꿈은 사라지지만(한 줄), 2번 때문에 여전히 넘쳐서 `콜`이 반만 보였음 → 둘 다 처리해야 완전 해결.

## 6. 조치 내역
- 근본 수정(`memo_type_badge.dart`의 배지 `Text`):
  ```dart
  Text(
    type.label,
    style: context.scaledTextStyle(AppTypography.caption12SB).copyWith(color: textColor),
    maxLines: 1,
    softWrap: false,                    // ① 한글 음절 줄바꿈 방지
    overflow: TextOverflow.clip,
    textScaler: TextScaler.noScaling,   // ② 시스템 글자배율 무시(고정크기 배지)
  )
  ```
- 디버그 계측(`assert(() {...debugPrint...}())`)은 확인 후 모두 제거.
- 조치 결과: 핫 리스타트 후 `아웃콜` 온전히 표시 확인.

## 7. 최종 상태
- 해결 여부: 해결 (사용자 "잘보인다" 확인).
- 남은 문제: 동일 패턴(고정폭 + 한글 라벨)의 다른 배지/칩(상병/오더 등)도 같은 증상 가능 → 점검 권장.

## 8. 학습 포인트
- 핵심 개념: ① 한글은 음절 단위 줄바꿈 가능 → 고정폭 단일라인 Text는 `softWrap:false` 필요. ② 시스템 textScaler가 고정크기 위젯을 넘치게 함 → `TextScaler.noScaling`. ③ 폭 측정 시 textScaler 누락 = 오판.
- 다음에 같은 증상이면 확인할 순서: 1) `debugPaintSizeEnabled`로 글자가 박스 밖으로 나가는지 눈으로 2) `MediaQuery.textScalerOf(context)` 값(>1?) 3) `TextPainter`에 textScaler 넣고 실측 vs 박스 내부폭.
- 👉 별도 학습 노트: [[2026-06-06-flutter-fixed-width-text-truncation]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 고정 크기 + 한글 라벨인 배지/칩 Text엔 `softWrap:false` + `textScaler:TextScaler.noScaling` 적용
  - [ ] 시각 검수는 **시스템 글자 크기를 키운 기기**에서도 1회 확인
- 추가하면 좋을 가드: 전역 textScaler 클램프 정책 논의(앱 자체 TokenScale과 시스템 배율 이중적용 방지). 단 본문 가독성/접근성과 트레이드오프라 PM 결정 필요.
- 자동화 후보: 고정폭 배지 라벨 폭을 textScaler 포함해 검증하는 위젯/골든 테스트.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: 고정 크기 칩/배지에서 끝 글자가 사라지거나 반만 보임. **Flex 넘침 노란줄 경고는 안 뜸**(Text 잘림은 경고 없음).
- 확인 절차: ① DevTools "Debug Paint" 켜서 글자가 테두리 밖으로 나가는지 ② Text의 `size`는 박스에 맞춰 잘려 보고되어 "딱 맞아 보이는 함정"이 있으니 신뢰하지 말 것 ③ `textScaler` 값 확인.
- 조치 절차: `softWrap:false` + `overflow:clip` + (고정크기면) `textScaler:TextScaler.noScaling`.
- 정상 확인 기준: 시스템 글자 110%+ 환경에서도 라벨 전체 표시.

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "내가 보기에는 MemoTypeBadege 에서 글자 사이즈가 안맞는거 같은데 확인해볼래ㅑ?"
- "Figma 에서 ellipsis 라는 설정은 어디서 확인해야하는거야? ... 피그마에서는 그냥 "메모2"로 보이는데? 그럼 피그마 설정이 뭔가 잘못됫다는거야? ... 이거 딱 가로가 31px 이라고 하는데 왜 안되지?"
- "자간 -1% 이거때문에 그런거 아니야?"
- "지금 아웃콜이 w 가 37.5 인데 뭔가 디버깅이 잘못 된 거 같아 피그마는 31 인데 이거 디버깅 방법을 자세히 고민해볼까?"
- "플러터 처음 하느 ㄴ사람도 이해할 수 있게 말해줄래? 아웃콜에서 콜 짜가 계속 짤려보이기는하는데,,,"
- "콜이 한줄로 표시되긴 했는데 텍스트가 37.5 를 넘는거 같아서 콜이 반만 보여"
- "잘보인다 근데 flutter debug 에서 도 ㅎ알 수 있나..?"
- "devtools를 열었는데 Text 에서는 어느 값을 봐야 하는지..? ... 그 위 container 에서는 어떤 값을 봐야하느지 다시 알려줘볼래"
- "text 는 size 가 15.0 으로 나오는걸, widtch 는 37.5 인게 나오고"
- "그리고 디버깅 코드들은 넣는다면 어디에 넣는거니"

### Claude가 되물은 확인 질문 + 답
- Q: 배지 폭/레이블 처리(고정 40px + ellipsis / 내용맞춤 hug / reviewer 시각대조) 중 어떻게 할까요?
  - A: "상황 설명 해줄래?" (→ 선택보다 원인 설명을 요청 → 계측 기반 디버깅으로 진행)

## 12. 한 줄 결론
> 고정폭 한글 배지의 끝 글자 잘림은 **① 한글 음절 softWrap + ② 시스템 textScaler** 두 원인이 겹친 것 — `softWrap:false` + `TextScaler.noScaling`으로 해결하고, 폭 측정 땐 textScaler를 반드시 포함해야 오판하지 않는다.
