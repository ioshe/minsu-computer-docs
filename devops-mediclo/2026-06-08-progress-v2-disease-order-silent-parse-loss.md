# 2026-06-08 · progress-v2 상병·오더가 화면에서 통째로 사라진 silent 파싱 누락

## 1. 개요
- 날짜: 2026-06-08
- 프로젝트 / 화면 / 모듈: devops-mediclo / 진료(consultation) Progress Note 카드 / `progress_v2_models.dart` (DTO 파싱)
- 이슈 유형: 버그 (API 연동 / JSON 파싱)
- 계층: API·네트워크 (응답 직렬화) + 클라이언트 (DTO 타입 선언)
- 최종 상태: 해결
- 중요도: 높음 (의료 데이터 — 상병/진단·오더가 화면에서 누락)

## 2. 환경
- OS / 기기 / 웹뷰: Flutter (태블릿 EMR 앱), 실서버 경로
- 앱 버전 / 빌드: develop 브랜치
- 브랜치 / 커밋: `develop` / 수정 커밋 `aae2014` `7a31e2a` `86ed8cc` `26528e1` `46d9d03`
- 관련 의존성: freezed, json_serializable, dio, retrofit
- API: `GET /v2/consultation/progress-v2`

## 3. 증상
### 관찰된 현상
- Progress Note 카드를 펼쳤을 때 **상병(상병명)이 표시되지 않고 `-` 로만** 보임.
- 에러/크래시는 전혀 발생하지 않음 (정상처럼 보임).
- 점검 결과 **오더(MEDICALITEMS)도 동일하게 누락**되고 있었음(사용자는 상병만 인지).

### 재현 절차
1. 실서버 모드로 환자(CUSTOMERID=500497) progress-v2 조회.
2. Progress Note 카드 펼침.
3. 상병칸/오더칸이 비어 `-` 표시.

### 에러 메시지 / 로그 (원문)
```text
(런타임 에러 없음 — 예외가 항목 단위 catch 로 조용히 삼켜짐)
실제 내부 예외: type 'String' is not a subtype of type 'num?' in type cast
```
서버 응답 일부 (타입에 주목):
```json
"DISEASELIST": [
  { "DISEASENAME": "기질성 기분[정동]장애, 우울 양상 동반",
    "DISEASETYPE": "1", "DISEASECODE": "F0630", "SUSPECT": "0", "SEQ": "0" }
],
"MEDICALITEMS": [
  { "ITEMNAME": "...", "CATEGORY": 2000, "SFLAG": "1|0",
    "UNITPRICE": "10000.00", "TOTALPRICE": "10000.00" }
]
```

## 4. 확인한 내용
- 매핑 경로 추적: `ConsultationProgressServiceImpl.fetchProgress` → `ProgressV2Response.fromJson` → `ConsultationRecord.fromJson`(DISEASELIST/MEDICALITEMS 루프) → `DiseaseDto/MedicalItemDto.fromJson`(생성된 `_$..FromJson`) → `ProgressV2Mapper.toProgressNote`(diseaseList→diagnoses).
- 생성 파서(`progress_v2_models.g.dart`) 확인:
  ```dart
  diseaseType: (json['DISEASETYPE'] as num?)?.toInt(),  // "1" as num? → throw
  suspect:     (json['SUSPECT'] as num?)?.toInt(),       // "0" as num? → throw
  category:    json['CATEGORY'] as String?,              // 2000(int) as String? → throw
  sFlag:       (json['SFLAG'] as num?)?.toInt(),         // "1|0" as num? → throw
  ```
- `ConsultationRecord.fromJson` 의 배열 루프가 **항목 단위 `try { ... } catch (_) {}`** 로 감싸져 있어,
  매 항목이 throw → **조용히 스킵** → `diseaseList`/`medicalItems` 가 빈 배열 → 매퍼가 빈 리스트 → UI 가 `-`.
- 매퍼(`ProgressV2Mapper._mapDiseases`)는 정상 — `d.diseaseName` 을 제대로 꺼냄. **범인은 DTO 파싱 단계 단 하나.**

## 5. 원인 분석
- **확정 원인:** 서버가 보내는 실제 wire 타입과 DTO의 `int?`/`String?` 선언이 불일치.
  json_serializable 의 생성 코드가 `as num?` / `as String?` **직접 캐스트**를 하는데, 값이 반대 타입(문자열 `"1"`, 정수 `2000`)이라 `TypeError(throw)` 발생.
  그 throw 를 배열 루프의 `catch (_)` 가 **로그 없이 삼켜** 데이터가 통째로 폐기됨(silent data loss).
- 부수 원인(별건): `UNITPRICE`/`TOTALPRICE` 가 `"10000.00"` 소수점 문자열 → 기존 `intFromDynamic`(int.tryParse)이 `null` 반환(크래시는 아님).

## 6. 조치 내역
커밋 5개로 분리:
- `aae2014` 상병: `DISEASETYPE`/`SUSPECT` 에 `fromJson: intFromDynamic` 적용(문자열/숫자 모두 안전 파싱).
- `7a31e2a` 오더: `CATEGORY` → `stringFromDynamic`(int→String 정규화), `SFLAG` → `String?` 로 타입 변경(`"1|0"` 은 본래 문자열), `EXCLAIM` 일관성 위해 `intFromDynamic`. `stringFromDynamic` 헬퍼 신규.
- `86ed8cc` 로깅: 세 catch(`data`/`DISEASELIST`/`MEDICALITEMS`)를 `catch (e)` + `if (kDebugMode) debugPrint('[progress-v2] ... parse skip: $e')` 로 전환(구조 유지, 가시성만 보강).
- `26528e1` 금액: `priceFromDynamic`(double 파싱 후 round) 헬퍼로 `UNITPRICE`/`TOTALPRICE`/STATEMENT 금액 6필드 적용.
- `46d9d03` chore: `import 'package:flutter/foundation.dart' show kDebugMode, debugPrint;` 로 축소(전체 import 시 freezed 가 DTO 마다 DiagnosticableTreeMixin 생성하던 코드 팽창 제거) + freezed 동기화.
- 조치 결과: `flutter analyze` clean, 생성 파서가 `intFromDynamic`/`stringFromDynamic`/`priceFromDynamic` 사용 확인. 상병·오더 정상 파싱.

## 7. 최종 상태
- 해결 여부: 해결 (상병·오더 표시 복구, 금액 소수점 파싱 정상화, silent failure → debug 로깅).
- 남은 문제: 없음. (실기기 화면 육안 확인은 사용자 몫으로 남김)

## 8. 학습 포인트
- 핵심 개념: **DTO 타입 불일치 + 항목 단위 `catch (_)` = silent data loss.** 캐스트 예외가 조용히 삼켜져 "에러 없이 데이터만 사라지는" 최악의 디버깅 난이도.
- 이번에 배운 점: 한 항목 throw 가 catch 로 스킵되면 리스트가 빈 채로 정상처럼 흐른다. 변환은 "제거"가 아니라 **경계(DTO) 한 곳에서 제대로**가 답.
- 다음에 같은 증상(데이터만 안 보임)이면 확인 순서: 1) 생성된 `_$..FromJson` 의 `as 타입` 직접 캐스트 vs 실제 응답 타입 대조 2) 배열 파싱 루프의 `catch` 가 throw 를 삼키는지 3) DTO `@JsonKey(fromJson:)` 변환기 유무.
- 👉 별도 학습 노트: [[2026-06-08-json-serializable-type-mismatch-silent-loss]], [[2026-06-08-freezed-foundation-import-diagnosticable]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 DTO 작성 시 **서버 실응답 샘플의 실제 타입**으로 필드 타입 결정(추측 금지).
  - [ ] 숫자/문자 혼입 가능 필드는 `intFromDynamic`/`stringFromDynamic`/`priceFromDynamic` 변환기 부착.
  - [ ] 배열 파싱 `catch` 는 `catch (e)` + `kDebugMode` 로깅으로(절대 `catch (_)` 침묵 금지).
  - [ ] 금액(원) 필드는 소수점 문자열 가능 → `priceFromDynamic`.
- 추가하면 좋을 가드/로깅: 파싱 skip 카운트를 계측해 "N건 누락" 을 모니터링에 노출.
- 자동화 후보: 실응답 픽스처로 DTO round-trip 단위 테스트(역직렬화 시 항목 수 보존 검증).

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: 에러는 없는데 특정 리스트(상병/오더 등)만 화면에서 비어 보임(`-`).
- 확인 절차: 생성 `_$..FromJson` 의 `as num?`/`as String?` 직접 캐스트 라인을 찾아, 실응답 JSON 의 해당 필드 타입과 대조. 배열 루프 `catch` 가 침묵하는지 확인.
- 조치 절차: 불일치 필드에 `fromJson:` 변환기 부착 또는 DTO 타입을 wire 타입에 맞춤 → `build_runner` 재생성 → `flutter analyze`.
- 정상 확인 기준: 변환기 적용된 생성 파서 확인 + 화면에 데이터 표시.

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "이거 현재 코드에서 매핑 잘되있니? 화면에서 상병명을 호출하지 못하는거 같아서 물어보는거임"
- "각각 커밋해주고"
- "현재' catch 항목이 // 파싱 실패 항목은 건너뜀 으로 되어 있는데 이게 맞는 구조일까 토론 나눠볼래 찬반 의견 sub agent 두고 나한테 결과 가져와바"
- "그냥 서버값을 그대로 코드에서 쓰는게 더 좋은 방식아닌가? 이것도 찬반해봐 ㅋ"
- "catch (_) → catch (e, st) + debugPrint('[progress-v2] disease parse skip: $e') (최소) 또는 계측 추가 / 이거 좋다 ㅇㅇ 나중에 ㄱrelease 때는 로그 안나오겠지?"
- "(B) 도메인 의미 변환을 아예 하지 말고 raw로 흘려라 → ❌ 반대 ... 올바른 답은 변환 제거가 아니라 변환을 경계 한 곳에서 제대로다 라는 말을 잘 이해못했어"

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 Claude 가 사실확인용으로 되물은 질문 없음 — 응답/코드가 충분해 바로 조치. 대신 진행 방향 선택지를 제시하면 사용자가 "둘다 수행"/"ㄱㄱ" 로 승인.)

## 12. 한 줄 결론
> 서버 wire 타입과 DTO 선언의 불일치가 캐스트 예외를 일으키고, 항목 단위 `catch (_)` 가 그걸 삼켜 상병·오더가 "에러 없이" 사라졌다 — 변환기를 경계 한 곳에 제대로 붙이고 catch 를 로깅으로 가시화해 해결.
