# Dart `implements` 시그니처 드리프트 → 런타임 NoSuchMethodError

> 한 줄 요약: 추상 인터페이스 타입으로 메서드를 호출하면, 구현체에만 추가된 `required` 인자는 컴파일 타임에 안 걸리고 런타임 `NoSuchMethodError` 로 터진다.

## 핵심
- 어떤 변수의 **정적 타입이 추상 인터페이스**(예: `PatientMemoService`)이면, 컴파일러는 그 변수로 하는 호출을 **인터페이스 시그니처 기준으로만** 검사한다.
- 실제로 들어있는 객체는 구현체(예: `RealPatientMemoService`)다. 구현체의 메서드에 인터페이스엔 없는 **`required` named 인자**가 추가돼 있으면, 호출은 그 인자 없이 통과 → 런타임 디스패치에서 인자가 모자라 `noSuchMethod` 가 호출된다.
- 즉 **"인터페이스 ↔ 구현체 시그니처 불일치(drift)"** 가 컴파일러를 우회해 런타임 크래시로 미뤄지는 구조다.
- `implements` 기반 다형성에서는 인터페이스 · 모든 구현체(실서버 · Mock) · 모든 호출측 시그니처가 **항상 한 덩어리로** 움직여야 한다.

## 언제 / 왜 쓰나
- Flutter/Dart 에서 서비스 계층을 `abstract class` + `implements`(실서버/Mock) 패턴으로 둘 때 자주 발생.
- 메서드에 인자를 하나 추가하면서 **구현체만** 고치고 인터페이스/다른 구현체/호출측을 빠뜨릴 때.
- 호출측이 인터페이스 타입을 통해 호출하므로(의도된 추상화), 빠뜨린 인자가 정적 분석에서 안 잡혀 **런타임까지 살아남는다.**

## 예시 (코드·명령)
```dart
// 인터페이스: authorName 없음
abstract class PatientMemoService {
  Future<PatientMemoDto> create({
    required int customerId,
    required String memo,
    // authorName 누락!
  });
}

// 구현체: authorName 을 required 로 추가 (drift)
class RealPatientMemoService implements PatientMemoService {
  @override
  Future<PatientMemoDto> create({
    required int customerId,
    required String memo,
    required String authorName, // ← 인터페이스엔 없는 required
  }) async { ... }
}

// 호출측: 정적 타입이 인터페이스라 authorName 없이도 "컴파일 통과"
final PatientMemoService svc = ref.read(patientMemoServiceProvider);
await svc.create(customerId: 1, memo: '...'); // 런타임에서 NoSuchMethodError
```

런타임 로그가 곧 진단서:
```text
NoSuchMethodError: Class 'RealPatientMemoService' has no instance method 'create' with matching arguments.
Tried calling: create(customerId: 1, memo: "...")                 // ← 실제 넘긴 인자
Found:        create({required ..., required String authorName})  // ← 구현 시그니처
```
`Tried calling` 과 `Found` 의 **차집합이 누락 인자**다.

해결: 인터페이스 · Mock · 실서버 · 모든 호출측에 인자를 일괄 반영 후 한 번에 검증.
```bash
flutter analyze lib/.../patient_memo_service.dart \
                lib/.../real_patient_memo_service.dart \
                lib/.../mock_patient_memo_service.dart \
                lib/.../current_jusojung_provider.dart
# → No issues found!
```

## 흔한 함정 / 헷갈리는 점
- "`implements` 했으니 시그니처가 다르면 컴파일 에러 나겠지" 라고 믿기 쉽지만, **호출측이 인터페이스 타입이면** 그 호출은 인터페이스만 보고 통과한다. 불일치는 **구현체를 직접 인스턴스로 호출할 때만** 정적으로 드러난다.
- named 인자라 순서 무관이라서 "인자 하나쯤" 빠진 걸 눈으로 못 알아챈다 → 로그의 `Tried/Found` 대비가 가장 빠른 단서.
- 구현체가 여러 개(실서버 + Mock)면 **둘 다** 고쳐야 한다. 하나만 고치면 mock 모드/실서버 모드 중 한쪽에서만 터진다.
- 함께 묻어오는 다른 버그를 놓치기 쉽다: 같은 작업에서 `update` 가 응답에 없는 필드를 `''` 로 채워 화면 값이 빈값으로 덮이는 잠재버그가 동반됐다(시그니처 정합화하면서 같이 발견·수정).

## 질문 기록

### 내가 던진 질문 (원문)
- "어느 계층에서 넣는게 좋을까? 뭐가 더 좋아보여?" (서비스/폼/모델 중 어디서 작성자명을 주입할지 — 결론: provider 계층)

### Claude가 되물은 확인 질문 + 답
- (이 학습 노트 관련 추가 확인 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-06-patient-memo-author-name-signature-drift]]
- 참고 링크: Dart 언어 사양 — method override / dynamic dispatch, `noSuchMethod`

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
