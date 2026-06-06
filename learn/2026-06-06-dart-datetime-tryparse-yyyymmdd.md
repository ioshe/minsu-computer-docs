# Dart `DateTime.tryParse` 함정 — 구분자 없는 `yyyyMMdd` 는 파싱 실패

> 한 줄 요약: `DateTime.tryParse("20260605")` 는 **null** 을 반환한다. 구분자 없는 `yyyyMMdd`/`yyyyMMddHHmmss` 는 ISO-8601 이 아니라서 직접 substring 파싱해야 한다.

## 핵심
- `DateTime.parse`/`tryParse` 는 **ISO-8601(또는 그 근사)** 만 받는다: `2026-06-05`, `2026-06-05T17:40:37`, `20260605T174037`(날짜+T+시간) 등.
- 그러나 **날짜만 있고 구분자가 없는** `"20260605"` 는 파싱 실패 → `parse` 는 `FormatException`, `tryParse` 는 **`null`**.
- 흔한 사고: `DateTime.tryParse(s) ?? DateTime.now()` 처럼 fallback 을 두면, 파싱이 조용히 실패하면서 **항상 현재 시각**으로 대체 → "날짜가 전부 오늘로 보인다" 버그.

## 언제 / 왜 쓰나
- 레거시/사내 API 가 날짜를 `yyyyMMdd`, 시각을 `yyyyMMddHHmmss` 같은 **구분자 없는 숫자 문자열**로 주는 경우가 많다(특히 EMR/금융/구 시스템).
- 이때는 `tryParse` 에 의존하지 말고 **명시적 substring 파싱** 헬퍼를 하나 만들어 SSOT 로 쓴다.

## 예시 (코드·명령)
```dart
// yyyyMMdd → DateTime (구분자 없음)
DateTime? parseYmd(String s) {
  if (s.length != 8) return null;
  final y = int.tryParse(s.substring(0, 4));
  final m = int.tryParse(s.substring(4, 6));
  final d = int.tryParse(s.substring(6, 8));
  if (y == null || m == null || d == null) return null;
  return DateTime(y, m, d);
}

// DateTime → yyyyMMdd (저장 시 하이픈 금지인 서버 대응)
String formatYmd(DateTime d) =>
    '${d.year}${d.month.toString().padLeft(2, '0')}${d.day.toString().padLeft(2, '0')}';

// ⚠️ 안티패턴: 조용한 fallback → 항상 오늘로 보임
final wrong = DateTime.tryParse('20260605') ?? DateTime.now(); // 항상 now!
```

## 흔한 함정 / 헷갈리는 점
- `"20260605T174037"`(중간에 T) 는 `tryParse` 가 **성공**하지만, `"20260605"`(날짜만) 와 `"20260605174037"`(T 없음) 은 **실패**한다. 포맷이 미묘하게 갈린다.
- 저장(역직렬화)도 짝을 맞춰야 한다. 조회는 substring 으로 잘 읽고, 저장은 `toIso8601String()`(하이픈 포함)으로 보내면 서버가 거부할 수 있다. **읽기/쓰기 포맷을 한 헬퍼 파일에 같이 둔다(SSOT).**
- `intl` 의 `DateFormat('yyyyMMdd').parse(s)` 도 대안이지만, 의존성 없이 쓸 거면 substring 이 가볍고 명확하다.

## 질문 기록
### 내가 던진 질문 (원문)
- (이 개념은 메모 이관 이슈 디버깅 중 파생 — 직접 던진 질문 없음)

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 기록할 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-06-memo-tab-realserver-migration]] (주소증 날짜가 항상 오늘로 표시된 버그의 근본 원인)
- 참고: `lib/core/utils/memo_date_utils.dart` (이 프로젝트의 SSOT 구현)

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
