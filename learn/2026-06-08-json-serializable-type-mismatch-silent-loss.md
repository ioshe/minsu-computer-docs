# json_serializable 타입 불일치 + catch(_) = silent 데이터 손실

> 한 줄 요약: 서버 wire 타입과 DTO 선언이 다르면 생성 파서의 `as 타입` 캐스트가 throw 하고, 항목 단위 `catch (_)` 가 그걸 삼켜 "에러 없이 데이터만 사라진다". 변환은 제거가 아니라 경계 한 곳에서 제대로.

## 핵심
- `@JsonSerializable`/freezed 가 만드는 `_$..FromJson` 은 기본적으로 **`json['X'] as num?`, `json['X'] as String?` 같은 직접 캐스트**를 쓴다. 값이 반대 타입이면 `TypeError` 를 던진다.
  - 예: 서버 `"DISEASETYPE": "1"`(문자열)인데 DTO `int? diseaseType` → `(json['DISEASETYPE'] as num?)` 에서 throw.
  - 예: 서버 `"CATEGORY": 2000`(정수)인데 DTO `String? category` → `as String?` 에서 throw.
- 리스트를 항목별로 `try { dto.add(X.fromJson(item)); } catch (_) {}` 로 감싸면, 위 throw 가 **로그 없이 스킵**되어 리스트가 빈 채로 정상처럼 흐른다 → UI 는 "그냥 비어 보임".
- 해결은 두 갈래이고 **필드 성격에 따라 다름**:
  - 진짜 그 타입이면 → `@JsonKey(fromJson: <안전변환기>)` 로 흡수(문자열 `"1"`도 int 로).
  - 본래 다른 타입이면 → **DTO 타입을 wire 타입에 맞춤**(예: `"1|0"` 플래그는 `String`).
- 원칙: **"변환을 하지 말자(raw로 흘리자)"가 아니라 "경계(DTO) 한 곳에서 제대로 변환".** 변환이 null 을 냈다면 변환이 나쁜 게 아니라 변환기가 어설픈 것(예: 소수점 문자열에 `int.tryParse`).

## 언제 / 왜 쓰나
- API 응답이 같은 필드를 때에 따라 int / "문자열" / "10000.00"(소수문자열) 로 보내는 백엔드와 붙을 때.
- "에러는 없는데 특정 리스트만 화면에서 비어 보인다" 증상을 만났을 때.

## 예시 (코드·명령)
```dart
// 안전 변환기 (한 파일에서 공유)
int? _parseIntSafe(dynamic v) {
  if (v == null) return null;
  if (v is int) return v;
  if (v is num) return v.toInt();
  if (v is String) return v.isEmpty ? null : int.tryParse(v);
  return null;
}
String? _parseStringSafe(dynamic v) {
  if (v == null) return null;
  if (v is String) return v.isEmpty ? null : v;
  return v.toString();              // 2000(int) → "2000"
}
int? intFromDynamic(dynamic v) => _parseIntSafe(v);
String? stringFromDynamic(dynamic v) => _parseStringSafe(v);
int? priceFromDynamic(dynamic v) {  // "10000.00" 소수 문자열용
  if (v == null) return null;
  if (v is int) return v;
  if (v is num) return v.round();
  if (v is String) return v.isEmpty ? null : double.tryParse(v)?.round();
  return null;
}

// 적용
@JsonKey(name: 'DISEASETYPE', fromJson: intFromDynamic) int? diseaseType,   // "1" → 1
@JsonKey(name: 'CATEGORY',    fromJson: stringFromDynamic) String? category, // 2000 → "2000"
@JsonKey(name: 'SFLAG') String? sFlag,                                       // "1|0" 은 타입 자체를 String 으로
@JsonKey(name: 'UNITPRICE',   fromJson: priceFromDynamic) int? unitPrice,    // "10000.00" → 10000

// 배열 파싱 catch — 침묵 금지
} catch (e) {
  if (kDebugMode) debugPrint('[progress-v2] disease parse skip: $e');
}
```
확인: `dart run build_runner build -d` 후 생성된 `_$..FromJson` 이 `as num?` 대신 `intFromDynamic(...)` 를 호출하는지 본다.

## 흔한 함정 / 헷갈리는 점
- `int.tryParse("10000.00")` → **null** (소수점 문자열 못 읽음). 금액은 `double.tryParse(...)?.round()` 써야 함.
- `catch (_)` 는 타입 에러든 OOM 이든 무차별 포획 → 최소한 `catch (e)` + `kDebugMode` 로깅. 구조(항목 격리)는 유지해도 됨, "침묵"만 제거.
- DTO 를 모두 `dynamic` 으로 두는 건 "raw로 흘리기" 의 끝 — 타입 안전성을 통째로 포기. 변환을 호출부 30곳에 흩뿌리게 됨.
- 매퍼(DTO→도메인)가 멀쩡해 보여도 의심하지 말 것: 데이터가 비면 보통 **그 앞 DTO 파싱 단계**가 범인.

## 질문 기록
### 내가 던진 질문 (원문)
- "그냥 서버값을 그대로 코드에서 쓰는게 더 좋은 방식아닌가? 이것도 찬반해봐 ㅋ"
- "올바른 답은 변환 제거가 아니라 변환을 경계 한 곳에서 제대로다 라는 말을 잘 이해못했어"

### Claude가 되물은 확인 질문 + 답
- (없음)

## 관련
- 발단이 된 이슈: [[2026-06-08-progress-v2-disease-order-silent-parse-loss]]
- 함께 본 함정: [[2026-06-08-freezed-foundation-import-diagnosticable]]
- 참고 링크: —

---
- 생성일: 2026-06-08
- 마지막 갱신: 2026-06-08
