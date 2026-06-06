# 2026-06-06 · 주소증 작성 시 NoSuchMethodError — interface/impl 시그니처 드리프트 + 작성자명 세션 연결

## 1. 개요
- 날짜: 2026-06-06
- 프로젝트 / 화면 / 모듈: devops-mediclo / 진료(consultation) · 주소증(jusojung) 작성 시트 / patient-memo 데이터 서비스
- 이슈 유형: 버그 (크래시성 런타임 에러 + 컴파일 에러)
- 계층: 클라이언트 (Dart 로직 · 서비스 추상화) — 일부 데이터/네트워크 인접
- 최종 상태: 해결
- 중요도: 높음 (저장 자체가 실패. EMR 작성자 귀속 정확성 문제 포함)

## 2. 환경
- OS / 기기 / 웹뷰: 안드로이드 태블릿 실기 (flutter run, `I/flutter (4752)` 로그)
- 앱 버전 / 빌드: develop 브랜치 디버그 빌드
- 브랜치 / 커밋: `develop` / `9728bfd`
- 관련 의존성·버전: Flutter 3.38.5 (stable) · freezed ^2.5.2 · freezed_annotation ^2.4.1 · flutter_riverpod ^2.5.1 · fpdart ^1.1.0

## 3. 증상
### 관찰된 현상
- 주소증 작성 후 AutoSave 가 동작하는 순간 저장 실패. 콘솔에 `NoSuchMethodError` 반복.
- 그와 별개로 IDE 에서 `current_jusojung_provider.dart:108` 에 컴파일 에러(`debugPrint` 미정의)가 떠 있었음.

### 재현 절차
1. 주소증 작성 시트를 열고 내용 입력.
2. AutoSave 타이머가 만료되어 `performCreate` → `createJusojung` → `_service.create(...)` 호출.
3. 실서버 구현(`RealPatientMemoService.create`)으로 디스패치되는 순간 크래시.

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
lib/.../current_jusojung_provider.dart:108:5: Error: The method 'debugPrint' isn't defined
for the type 'CurrentJusojungList'.
    debugPrint('[MyDebug] jusojung show(=${jusojung})');

I/flutter ( 4752): [AutoSave] save failed: NoSuchMethodError: Class 'RealPatientMemoService'
has no instance method 'create' with matching arguments.
I/flutter ( 4752): Receiver: Instance of 'RealPatientMemoService'
I/flutter ( 4752): Tried calling: create(category: ..., customerId: 500497,
   memo: "...", memoDate: "2026-06-05", scheduleId: 125182, top: true)
I/flutter ( 4752): Found: create({required int customerId, required int scheduleId,
   required MemoCategory category, required String memo, required String memoDate,
   required String authorName, bool top}) => Future<PatientMemoDto>
I/flutter ( 4752): #1  CurrentJusojungList.createJusojung
   (package:mediclo/.../current_jusojung_provider.dart:112:32)
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점:
  - `patient_memo_service.dart` (추상 인터페이스), `real_patient_memo_service.dart`(실서버), `mock_patient_memo_service.dart`(Mock), 호출측 `current_jusojung_provider.dart` 4곳 시그니처 대조.
  - `flutter analyze <4파일>` 로 각 단계 검증.
- 결과 해석:
  - 로그의 `Tried calling: ...(authorName 없음)` vs `Found: ...(required String authorName)` 대비가 곧 원인. 호출측은 `authorName` 없이 불렀는데, 실제 구현은 그걸 **required** 로 요구.
  - 추상 인터페이스 `PatientMemoService.create` 에는 `authorName` 이 **없었고**, 실서버 구현에만 `required String authorName` 가 추가돼 있었다 → 인터페이스와 구현의 시그니처가 어긋남(drift).
  - 호출측 `_service` 의 정적 타입은 인터페이스라, 컴파일러는 인터페이스 시그니처(=authorName 없음) 기준으로 호출을 통과시킴. 실제 객체는 실서버 구현이라, 런타임 디스패치에서 required 인자 누락 → `noSuchMethod`.

## 5. 원인 분석
- **확정 원인:** `PatientMemoService`(추상) 와 `RealPatientMemoService`(구현)의 `create` 시그니처 드리프트.
  구현에만 `required String authorName` 가 추가됐고 인터페이스·Mock·호출측에는 반영되지 않음.
  호출측은 인터페이스 타입으로 호출하므로 누락이 **컴파일 타임에 안 잡히고 런타임에 터짐**.
- **부수 원인(컴파일 에러):** `createJusojung` 안에 임시 디버그 로그 `debugPrint('[MyDebug] ...')` 가 남아 있었는데
  `package:flutter/foundation.dart` 가 import 되지 않아 `debugPrint` 미정의 에러.
- **추가로 드러난 잠재 버그:** 실서버 `update` 는 응답에 작성자명이 없어 `emplName: ''` 로 반환 → 주소증 수정·핀 토글만 해도
  화면 작성자명이 빈값으로 덮이는 문제가 잠복해 있었음.

## 6. 조치 내역

### (a) 컴파일 에러 — 임시 로그 제거
- `createJusojung` 의 `[MyDebug]` 디버그 로그를 삭제(원래 임시 추적용). foundation import 도 불필요해져 정리.

### (b) 시그니처 통일 — create 에 authorName 전파 (근본 수정)
- `patient_memo_service.dart` 인터페이스 `create` 에 `required String authorName` 추가.
- `mock_patient_memo_service.dart` `create` 에 동일 추가 + `emplName: authorName` 로 반영(기존 `'김의사'` 하드코딩 제거).
- 호출측 `createJusojung` 은 **폼 값이 아니라 로그인 세션**에서 작성자명을 읽도록:
  ```dart
  // 작성자명은 폼 입력이 아니라 로그인 세션에서 가져온다 (EMR 귀속 정확성).
  final authorName = ref.read(authProvider).userName ?? '';
  final dto = await _service.create(..., authorName: authorName, top: jusojung.isPinned);
  ```
  - `authProvider`(=`lib/core/auth/auth_state.dart`)의 `AuthState.userName`(= `userInfo.emplName`) 사용.
  - fallback 은 `'김의사'` 같은 날조 대신 **빈 문자열** — EMR 에서 "그럴듯한 가짜 작성자"보다 "정직한 불명"이 안전.
- 폼(`write_jusojung_sheet.dart`)의 `buildRecord()` 하드코딩 `authorName: '김의사'` → `''` placeholder 로 정리(어차피 provider 가 덮어씀).

### (c) update 경로도 동일 정합화 + 잠재버그 수정
- 인터페이스/실서버/Mock `update` 에 `required String authorName` 추가.
- 실서버 `update` 의 `emplName: ''` → `emplName: authorName` (수정/핀 토글 시 작성자명 사라지던 버그 해결).
- **정책 결정: 수정 시 작성자명은 "원작성자 보존"** (수정자 ≠ 작성자). create 는 로그인 사용자로 채우지만,
  update 는 기존 항목의 작성자를 유지.
  - `updateJusojung`: 폼 record 의 authorName 은 placeholder 라 신뢰하지 않고, `_items` 에서 원본을 찾아 전달.
    ```dart
    final original = _items.firstWhere((j) => j.id == jusojung.id, orElse: () => jusojung);
    ... authorName: original.authorName ...
    ```
  - `togglePin`: 핀 토글도 update 경로 → `target.authorName` 전달.

### 조치 결과
- `flutter analyze` 대상 4파일 모두 `No issues found!`.

## 7. 최종 상태
- 해결 여부: 해결 (정적 분석 통과). 런타임 동작은 mock 모드 작성→수정→핀토글로 작성자명 유지 확인 권장(미실행).
- 남은 문제 / 추가 확인 필요:
  - 실서버 모드에서 `authProvider.userName` 이 실제 의사명으로 채워지는지(서버 `EMPLNAME`) 1회 확인.
  - 동일 시그니처 드리프트가 다른 patient-memo 호출측(메모/기타 시트)에도 없는지 점검.

## 8. 학습 포인트
- 핵심 개념: **추상 인터페이스로 호출하면, 구현체에만 추가된 required 인자는 컴파일 타임에 안 걸리고 런타임 NoSuchMethodError 로 터진다.**
- 이번에 배운 점:
  - `implements` 패턴에서 인터페이스/Mock/실서버 시그니처는 **한 덩어리로 같이** 움직여야 한다.
  - NoSuchMethodError 로그의 `Tried calling:` vs `Found:` 대비가 곧 누락 인자를 가리키는 결정적 증거.
  - EMR 도메인에선 작성자 귀속 fallback 을 "그럴듯한 가짜"로 채우면 안 된다(의료법적 귀속). 불명은 불명으로.
- 다음에 같은 증상이면 확인할 순서:
  1) NoSuchMethodError 의 `Tried calling` 인자 vs `Found` 시그니처 차이부터 본다.
  2) 호출측 `_service` 의 **정적 타입(인터페이스)** 과 **실제 객체(구현체)** 시그니처를 대조.
  3) 인터페이스/Mock/실서버 3종 + 모든 호출측을 `flutter analyze` 로 한꺼번에 검증.
- 👉 별도 학습 노트: [[2026-06-06-dart-implements-signature-drift-nosuchmethod]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 서비스 메서드 시그니처 변경 시 **인터페이스 → 실서버 → Mock → 모든 호출측**을 동시에 수정했는가
  - [ ] 변경 후 관련 4종 파일을 한 번에 `flutter analyze` 했는가
  - [ ] 임시 `[MyDebug]`/`debugPrint` 추적 로그를 커밋 전에 제거했는가
- 추가하면 좋을 테스트 / 가드 / 로깅:
  - Mock 서비스 기반 단위 테스트로 create/update 호출 시그니처 회귀 방지(인자 누락 시 컴파일 실패 유도).
- 자동화 후보:
  - pre-commit 에서 `[MyDebug]` 같은 임시 로그 패턴 grep 차단.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `NoSuchMethodError: Class 'XxxServiceImpl' has no instance method '...' with matching arguments`.
- 확인 절차: 로그의 `Tried calling`(실제 넘긴 인자) ↔ `Found`(구현 시그니처) 비교 → 누락/추가된 named 인자 식별 → 인터페이스에 그 인자가 있는지 확인.
- 조치 절차: 누락 인자를 인터페이스·Mock·실서버·호출측에 일괄 반영 후 `flutter analyze`.
- 정상 확인 기준: 관련 파일 `No issues found!` + 실제 작성/수정 1회 성공.

## 11. 질문 기록

### 내가 던진 질문 (원문)
- "form d을 받는 부분은 어디야? 그니까 form에서 어느 데이터를 받아올껀지에 대한 부분...?"
- "주소증은 이게 로그인한 의사 이름 불러오는건데 현재 provider 에서 authorNmame d을 가지고 있는 부분이 있니?"
- "어느 계층에서 넣는게 좋을까? 뭐가 더 좋아보여?"
- "jusojung 이 작성자 이름을 담아야 하나? 아 담긴 담아야 하는구나, 그럼 빈 문자열이 좋을려나? 뭐가 좋아보여...?"
- "update 부분도 고쳐줄래 ㅋㅋ"
- "authorName ->< userName" (※ 범위 확인 질문을 띄웠으나 사용자가 취소 → 미진행)

### Claude가 되물은 확인 질문 + 답
- Q: `authorName` → `userName` 변경을 어느 범위로(지역변수/서비스 파라미터/모델 필드) 적용할까?
  - A: ❓ 미답 (질문 도중 사용자가 도구 사용을 취소함. 실제 rename 미수행)
- Q: create 적용 시 `userName` null fallback 을 빈 문자열로 둘지 `'김의사'`로 둘지?
  - A: 빈 문자열(`?? ''`) 채택 — EMR 귀속 정확성 근거.

## 12. 한 줄 결론
> 인터페이스 타입으로 호출하면 구현체에만 추가된 required 인자는 런타임 NoSuchMethodError 로 드러난다 — `implements` 3종 + 호출측 시그니처는 항상 한 덩어리로 같이 고친다.
