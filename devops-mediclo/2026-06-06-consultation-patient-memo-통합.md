# 2026-06-06 · 진료실 주소증/메모/기타 — 환자 데이터 하드코딩 제거 + patient-memo 단일화 리팩토링

> 이 문서는 "버그를 고치다가 구조 리팩토링까지 간" 한 세션의 기록이다.
> Flutter/Riverpod 가 처음인 사람이 **이 프로젝트의 데이터 흐름 구조**를 같이 공부할 수 있게,
> 평소 dev-log 보다 훨씬 길고 친절하게 썼다. 개념 자체는 별도 학습 노트로 분리했다 →
> [[2026-06-06-flutter-진료실-데이터흐름-구조]]

---

## 1. 개요
- 날짜: 2026-06-06
- 프로젝트 / 화면 / 모듈: devops-mediclo / 진료실(consultation) / 차트 우측 메모 영역(주소증·메모·기타 탭)
- 이슈 유형: 버그(잘못된 데이터 표시) + 그로 인한 구조 리팩토링
- 계층: **클라이언트(상태관리·데이터 계층)** — 서버 코드는 안 건드림
- 최종 상태: 해결 (커밋 `eaf4ec2`)
- 중요도: 높음 (환자가 뒤바뀐 데이터를 볼 수 있는 문제 → EMR 에서 치명적)

## 2. 환경
- 대상: 태블릿(iPad/Galaxy Tab) EMR 앱, Flutter (Dart ^3.10.4)
- 상태관리: flutter_riverpod + riverpod_generator
- 불변 모델: freezed
- 결과타입: fpdart `Either<Failure, T>`
- 브랜치 / 커밋: develop / 결과 커밋 `eaf4ec2`
- 실행 모드: 주로 **Mock 모드**(가짜 데이터)로 검증. 실서버 endpoint 는 일부만 존재.

## 3. 증상

### 관찰된 현상
1. 진료실에 **어떤 환자로 들어가도 주소증 탭에 항상 같은 사람("500497")의 데이터**가 떴다.
2. **기타(Etc) 탭이 안 불러와졌다** — 환자에 따라 빈 화면.

### 재현 절차
1. Mock 모드로 로그인 → 현황판에서 아무 환자 차트나 탭 → 진료실 진입.
2. 주소증 탭: 어떤 환자든 동일 내용 표시(원래는 환자별로 달라야 함).
3. 기타 탭: 대부분의 환자에서 "표시할 기타 항목이 없습니다."

### 핵심이 된 코드 (원문)
```dart
// lib/features/consultation/presentation/providers/current_jusojung_provider.dart (수정 전)
// TODO(연결): 실제 환자 컨텍스트 provider 에서 customerId 가져오기
//   예) ref.watch(currentPatientProvider).customerId
int get _customerId => 500497;   // ← 모든 환자가 이 값으로 조회됨
```

## 4. 확인한 내용

진료실의 "환자가 누구인가"는 어떻게 정해지는지 데이터 흐름을 거꾸로 추적했다.

- 라우트: `/consultation/:scheduleId` — 진료실은 **`scheduleId`(예약 ID) 하나만** 들고 진입한다.
  (`lib/config/routes/app_router.dart` 의 `consultation` 라우트)
- `customerId`(환자 ID)는 라우트로 안 넘어온다. 대신 **현황판 데이터에서 lookup** 한다:
  `statusBoardProvider.allSchedules` 에서 `scheduleId` 가 같은 예약을 찾아 → 그 예약의 `patient.customerId`.
- 이 변환을 캡슐화한 공용 헬퍼가 이미 있었다:
  ```dart
  // lib/features/consultation/presentation/providers/current_patient_provider.dart
  int? resolveCustomerIdFromSchedule(List<ScheduleModel> allSchedules, String scheduleId) { ... }
  ```
- 그런데 탭마다 이 헬퍼를 쓰는 정도가 제각각이었다:
  | 탭 | customerId 출처 | 상태 |
  |----|----------------|------|
  | 증상경과 | `resolveCustomerIdFromSchedule` 사용 | 정상 |
  | 메모 | 같은 로직을 인라인으로 직접 작성 | 정상(중복) |
  | **주소증** | **`=> 500497` 상수** | **버그** |

기타가 안 뜬 건 별개의 원인이었다. 각 도메인 **Mock 픽스처의 customerId 가 서로 안 맞았다.**
- 현황판 스케줄 픽스처: `customerId` 10001~10011
- 기타(Etc) 픽스처: 10001/10002/10003 만 존재
- 주소증/메모(patient-memo) 픽스처: **500497 만** 존재

→ 주소증이 500497 하드코딩일 땐 "우연히" 픽스처와 맞아 데이터가 보였던 것이고,
하드코딩을 풀어 진짜 customerId(10001~)로 조회하자 **주소증/메모 픽스처(500497)와 안 맞아 빈 화면**이 됐다.
기타는 처음부터 10001~3 외 환자에선 빈 화면이었다.

## 5. 원인 분석

- **확정 원인 1 (주소증):** `_customerId` 가 상수 `500497` 로 박혀 있어, 선택한 환자와 무관하게 한 사람 데이터만 조회.
- **확정 원인 2 (기타 안 뜸):** Mock 픽스처의 `customerId` 가 도메인마다 달라(현황판 10001~, patient-memo 500497, etc 10001~3) 진짜 customerId 로 조회 시 매칭 실패.
- **부수 발견:** 기타 탭은 별도 `EtcService`(실서버 구현 없이 Mock 만)로 돌아가고 있었는데, 알고 보니 **기타도 주소증/메모와 같은 단일 엔드포인트** `/v2/consultation/patient-memo` 로 처리되는 종류였다(`MemoCategory.etc` → `MEMOTYPE=4`). 즉 별도 서비스가 불필요한 중복이었다.

## 6. 조치 내역

작업은 사용자와 대화하며 4단계로 진행됐다. 각 단계가 왜 필요했는지가 학습 포인트다.

### 6-1. 주소증 하드코딩 제거 (근본 수정)
`current_jusojung_provider.dart` 에서 상수를 지우고, `build(scheduleId)` 안에서 동적 도출.
```dart
// 수정 후
int _customerId = 0; // build() 에서 채움

@override
Future<List<Jusojung>> build(String scheduleId) async {
  final allSchedules = ref.watch(statusBoardProvider).allSchedules;
  final customerId = resolveCustomerIdFromSchedule(allSchedules, scheduleId);
  if (customerId == null) { _customerId = 0; _items = []; return _items; } // 환자 못 찾으면 빈 목록
  _customerId = customerId;
  // ... 이후 patient-memo fetch
}
```

### 6-2. Mock 픽스처 customerId 일관성 정렬
CLAUDE.md 규칙("customerId=10001 은 모든 도메인에서 동일 환자")에 맞춰 patient-memo 픽스처의 `500497` → `10001` 로 통일. 이제 첫 환자(10001)로 들어가면 주소증·메모·기타가 **모두** 뜬다.

### 6-3. 기타(Etc) → patient-memo 단일 엔드포인트로 통합 ("방법 A")
- 삭제: `etc_service.dart`, `mock_etc_service.dart`, `etc_fixtures.dart` (별도 서비스 일체)
- 기타 조회/작성/수정/삭제를 `patientMemoServiceProvider` + `MemoCategory.etc` 로 위임.
- `PatientMemoDto`(서버 응답 원형) → `Etc`(화면 모델) 변환 매퍼 `toEtc()` 추가.
- 기타 픽스처는 patient-memo Mock 안으로 `MEMOTYPE=4` 항목으로 이관.

왜 통합? **백엔드가 진짜로 엔드포인트 하나**이기 때문. "서비스 1개 = 엔드포인트 1개" 원칙상,
같은 URL 을 때리는 서비스를 3개로 쪼개면 Dio 호출·에러 매핑·디코드 로직이 3벌 복제된다.

### 6-4. create/update 시그니처 리팩토링 — `PatientMemoWrite` 값객체 ("옵션 1")
통합하다 보니 `create/update` 에 도메인 전용 파라미터가 평평하게 쌓였다
(`memoType`/`callType` 은 메모 전용, `color` 는 기타 전용). 주소증 호출자도 안 쓰는 `color` 를 보게 되는 **인터페이스 오염(ISP 위반)**. 실제로 기타 `color` 를 추가했더니 주소증/메모 시그니처까지 바뀌었다.

→ 변형 속성을 값객체로 묶고 도메인 팩토리로 의도를 표현:
```dart
class PatientMemoWrite {
  const PatientMemoWrite._({required this.category, this.memoType = 0, this.callType = 0, this.color = ''});
  final MemoCategory category; final int memoType; final int callType; final String color;
  const PatientMemoWrite.jusojung() : this._(category: MemoCategory.jusojung);
  const PatientMemoWrite.memo({int memoType = 0, int callType = 0})
      : this._(category: MemoCategory.memo, memoType: memoType, callType: callType);
  const PatientMemoWrite.etc({String color = ''}) : this._(category: MemoCategory.etc, color: color);
}
// 호출측: service.create(..., write: PatientMemoWrite.etc(color: token));
```
호출처 8곳(주소증3·메모3·기타2) 전부 이 형태로 교체.

### 6-5. `Etc.isUnread` → `isNew` 통일
`Etc` 만 N 배지 플래그를 `isUnread` 라 불렀는데, 메모/주소증의 `isNew`(= "오늘 작성") 와 **완전히 같은 의미**였다. 이름을 `isNew` 로 통일(freezed 모델이라 build_runner 재생성).

### 조치 결과
- `flutter analyze`: 변경 파일 이슈 0 (잔여 7건은 이번 작업과 무관한 사전 존재 항목).
- Mock: 첫 환자(10001)에서 주소증·메모·기타 모두 표시, 환자별로 다른 데이터 확인.
- 커밋 `eaf4ec2` (15 files, +501/−465).

## 7. 최종 상태
- 해결 여부: **해결.** 환자별로 올바른 데이터가 조회되고, 기타 탭이 동작하며, 데이터 계층이 단일 서비스로 통일됨.
- 남은 문제 / 추가 확인 필요:
  - 실서버 기타 `COLOR` 필드 포맷이 "정수 문자열"이 아니면 `toEtc`/`etcColorToken` 매퍼만 조정 필요.
  - 메모 탭·consultation_page 의 인라인 customerId 도출이 아직 공용 헬퍼를 안 씀(동작은 정상, 일관성 개선 여지).

## 8. 학습 포인트
- 핵심 개념:
  1. 진료실은 `scheduleId` 만 들고 오고 `customerId` 는 **현황판에서 lookup** 한다 (도출 함수 1곳에 캡슐화).
  2. "서비스 = 엔드포인트" 1:1 — 백엔드가 단일 endpoint 면 클라 서비스도 하나로 두는 게 중복이 적다.
  3. 선택 파라미터가 쌓이면 **값객체 + 도메인 팩토리**로 시그니처 오염(ISP)을 막는다.
  4. Mock 픽스처는 **도메인 간 ID 일관성**이 생명. 안 맞으면 "버그처럼 보이는데 사실 데이터 문제".
- 다음에 같은 증상(데이터가 안 뜨거나 엉뚱하게 뜸)이면 확인 순서:
  1) 화면이 쓰는 `customerId`(또는 키)가 **어디서 오는지** 역추적
  2) 그 값이 **Mock/서버 데이터의 키와 실제로 일치**하는지
  3) Provider(`family`) 키가 매번 같은지 / 무효화(invalidate)는 되는지
- 👉 구조 개념은 별도 학습 노트로: [[2026-06-06-flutter-진료실-데이터흐름-구조]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 탭/Provider 추가 시 `customerId` 는 **반드시 `resolveCustomerIdFromSchedule` 사용**(상수 금지)
  - [ ] 새 Mock 픽스처는 `customerId=10001` 등 **공통 환자 ID 규약**을 따른다
  - [ ] 새 서비스 만들기 전, **기존 엔드포인트로 처리 가능한지** 먼저 확인(불필요한 평행 서비스 금지)
  - [ ] create/update 에 도메인 전용 파라미터를 더할 땐 **값객체로** 받을지 검토
- 추가하면 좋을 가드:
  - 디버그 빌드에서 `customerId == 0`(미도출) 조회 시 경고 로그
  - 위젯 테스트: "서로 다른 scheduleId → 서로 다른 데이터" 검증

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: 환자를 바꿔도 같은 데이터 / 특정 탭만 빈 화면
- 확인 절차:
  1. 해당 탭 Provider 의 키(`customerId`) 출처 확인 — 상수/인라인/헬퍼?
  2. `[MOCK] ...fetch(cust=...)` debugPrint 로 실제 넘어간 customerId 확인
  3. 그 customerId 가 픽스처에 존재하는지 (`grep customerId fixtures`)
- 조치 절차: 도출 로직을 헬퍼로 통일 / 픽스처 ID 정렬
- 정상 확인 기준: 환자 2명 이상에서 각자 다른 데이터가 뜸

## 11. 질문 기록

### 내가 던진 질문 (원문)
- "와 같은 api가 있어 차트를 누르면 진료실로 들어가는 api인데 여기서 선택한 환자의 메모를 불러오게 해야해 지금 하드 코딩되어 있을텐데 바꿀 준비만 해줄래? 계획 잡아봐"
- "ㅇㅇ 고쳐주셈 근데 어디서 customerid 받아오는지느 ㄴ알아?"
- "무슨 말인지 잘모르겠네?"
- "ㄴ 다른 코드에서도 하드코딩 ㅣㅇㅆ는 부분 확인해줘"
- "기타가 안불러와지는걸"
- "etc service 가 아직 실서비스에 연동된게 아닌가"
- "Pm아 기타 연동하는 방법 알고 있니?"
- "방법 A 하고 제거 ㅇㅇ"
- "지금 patinetmemo가 3개의 api를 통합하느라 덕지덕지 붙은 느낌인데 헬퍼함수를 두고 가각 필요한 기능ㅎ만 추가해서 쓰는 방식이 나을려나? 어떻게 생각해 찬반 토론해볼래? sub agnet 둬서"
- "1번 어떻게 하게[ㅆ다는건지 예시 코드 보여줄 수 이썽?"
- "dㅇㅇㅇ 바꿔주고"
- "커밋도 하고"
- "isUnread sms isNew랑 같은 거 같은데? 꼭 있어야 하니" / "그니까 IsNew로 똑같이 가면 안되냐는거지"

### Claude가 되물은 확인 질문 + 답
- Q: customerId 도출 실패 시 정책(빈 목록 vs 에러)과 두 번째 Mock 환자 픽스처를 추가할까?
  - A: (그대로 진행 지시) 빈 목록 폴백으로 처리, 픽스처는 이후 10001 정렬로 해소.
- Q: 기타 연동을 방법 A(patient-memo 통합) vs 방법 B(EtcServiceImpl 신규) 중 무엇으로?
  - A: "방법 A 하고 제거" — 통합 + 기타 전용 서비스 제거.
- Q: 시그니처 정리(옵션 1 값객체)를 적용할까?
  - A: "ㅇㅇㅇ 바꿔주고" — 적용.

## 12. 한 줄 결론
> 진료실은 `scheduleId`만 들고 오고 `customerId`는 현황판에서 도출한다 — 이 원칙을 어긴 상수 하나가 모든 환자에게 같은 데이터를 보여줬고, 고치는 김에 기타를 patient-memo 단일 서비스로 통합하고 값객체로 시그니처를 정리했다.
