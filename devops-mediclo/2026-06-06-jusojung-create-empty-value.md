# 2026-06-06 · 주소증 신규 작성 후 화면에 빈 값으로 표시되는 버그

## 1. 개요
- 날짜: 2026-06-06
- 프로젝트 / 화면 / 모듈: Mediclo · 상담(consultation) · 주소증(jusojung) 작성 시트 / 목록
- 이슈 유형: 버그 (API 연동 — 응답 파싱)
- 계층: API·네트워크 (서버 mutation 응답 형태 ↔ 클라이언트 모델 매핑 불일치)
- 최종 상태: 해결
- 중요도: 높음 (작성한 내용이 화면에서 사라져 보이는 핵심 기능 결함)

## 2. 환경
- OS / 기기 / 웹뷰: Android 태블릿 (Flutter 앱, 실기기 로그 기준)
- 앱 버전 / 빌드: 1.0.0+1
- 브랜치 / 커밋: develop / 진단 시점 9728bfd → 정본화 후 8053ea3 (Either+Failure 이관 P2)
- 관련 의존성·버전: dio, flutter_riverpod + riverpod_generator, fpdart(Either)

## 3. 증상
### 관찰된 현상
- "주소증 작성하기" → 새 주소증(예: "상용구")을 입력·저장하면 POST 는 성공(200)하지만,
  방금 추가된 카드가 **내용 없이 빈 값**으로 목록에 나타남.
- 작성자명 영역도 빈칸으로 표시됨.
- 화면을 다시 불러오면(fetch) 정상적으로 내용이 보임 → "저장 직후"에만 빈 값.

### 재현 절차
1. 환자 상담 화면에서 주소증 작성 시트 열기.
2. 본문에 텍스트 입력(자동저장 debounce 동작).
3. 시트를 닫으면 목록에 방금 항목이 추가됨 → **내용이 빈 카드**로 보임.

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
# GET (작성 전 목록 조회 — 정상 데이터 존재)
uri: .../v2/consultation/patient-memo?filter={"CUSTOMERID":500497,"type":0}&page={"offset":0,"limit":30}
Response: {"status":"SUCCESS","code":200,"data":[{"MEMOID":18015,...,"MEMO":"상용구","MEMODATE":"2026-06-",...}, ...]}

# POST (신규 작성)
[Draft] saved: jusojung_125182
[AutoSave] jusojung create
uri: .../v2/consultation/patient-memo  method: POST
data: {CUSTOMERID: 500497, MEMO: 상용구, MEMODATE: 2026-06-05, MEMOTYPE: 1, CALLTYPE: 0, SCHEDULEID: 125182, TOP: 1, COLOR: }

# POST 응답 — 식별자만 반환, MEMO/MEMODATE/EMPLNAME 없음
Response: {"status":"SUCCESS","code":200,"data":{"fieldCount":0,"affectedRows":1,"insertId":18016,"info":"","serverStatus":2,"warningStatus":1,"MEMOID":18016}}
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점:
  - POST 응답 body 직접 확인 → `data` 안에 `insertId`/`MEMOID`만 있고 **MEMO/MEMODATE/TOP/EMPLNAME 없음**.
  - `RealPatientMemoService.create()` 가 그 응답을 `PatientMemoDto.fromJson(_unwrap(res.data))` 로 그대로 파싱.
  - `PatientMemoDto.fromJson` 은 키가 없으면 기본값(`memo: ''`, `memoDate: ''`, `emplName: ''`, `top: 0`)으로 채움.
- 결과 해석:
  - 생성 응답으로 만든 DTO 는 `memoId` 만 유효하고 나머지가 비어 있음 →
    provider 의 `createJusojung` 에서 `_toJusojung(dto)` 가 **content 빈 Jusojung** 을 만들어 목록에 추가 →
    `JusojungCard` → `NoteCard(content: '')` 로 빈 카드 렌더.
  - Mock 서비스는 입력값으로 전체 DTO 를 만들어 반환했기 때문에 Mock 모드에서는 정상 → 실서버에서만 재현된 이유.

## 5. 원인 분석
- **확정 원인:** 서버의 생성(POST)·수정(PUT) 응답이 **mutation 결과 메타(insertId/affectedRows)와 식별자(MEMOID)만** 반환하고
  메모 본문(MEMO/MEMODATE/TOP/EMPLNAME)을 포함하지 않는데, 클라이언트가 그 응답을 곧바로 도메인 DTO 로 파싱해
  화면에 표시할 모델로 사용했기 때문. 비어 있는 필드가 그대로 UI 에 전달됨.
- 가능성 있는 원인: update(PUT) 도 동일 응답 형태일 가능성이 커 같은 버그 잠재(선제 수정함).

## 6. 조치 내역
- 수행한 조치 (핵심):
  - `create()`/`update()` 가 **응답에서는 식별자(MEMOID, 없으면 insertId)만** 취하고,
    나머지 필드는 **요청에 보낸 입력값으로 DTO 를 직접 구성**해 반환하도록 변경.
  - 작성자명을 채우기 위해 서비스 `create`/`update` 시그니처에 `required String authorName` 추가 →
    인터페이스·Mock·호출부(provider)까지 전파. 호출부는 `authorName: jusojung.authorName` 로 입력값(작성자) 보존.
  - 핵심 로직(현재 통합 파일 `patient_memo_service.dart`):
    ```dart
    // 서버는 생성 결과로 MEMOID(insertId)만 돌려주므로, 나머지 필드는 입력값으로 구성
    final newId = (data['MEMOID'] as num?)?.toInt() ?? (data['insertId'] as num?)?.toInt() ?? 0;
    return PatientMemoDto(
      memoId: newId, customerId: customerId, scheduleId: scheduleId,
      memo: memo, memoDate: memoDate, memoType: category.memoType,
      callType: 0, top: top ? 1 : 0,
      emplName: authorName, // 응답에 작성자명 없음 — 호출측이 보존한 원작성자 사용
    );
    ```
- 임시 우회책(workaround): (없음 — 근본 수정으로 처리)
- 근본 수정(root fix): 위와 같이 mutation 응답을 식별자 소스로만 쓰고 모델은 입력값으로 재구성.
- 조치 결과: 작성 직후 내용·날짜·고정·작성자명 모두 정상 표시. `flutter analyze` 에러 없음(자동생성 deprecated 경고만).
- 후속: 이후 커밋 8053ea3 에서 서비스가 `Either<PatientMemoFailure, T>` 표준으로 이관되며 단일 파일로 통합됨.
  픽스 로직은 그대로 유지됨([patient_memo_service.dart] create 189~207, update 241~251 부근).

## 7. 최종 상태
- 해결 여부: 해결.
- 남은 문제 / 추가 확인 필요:
  - `WriteJusojungSheet.buildRecord()` 의 `authorName: '김의사'` 는 **하드코딩 임시값**.
    실제 로그인 사용자명으로 교체 필요(세션/로그인 provider 주입).
  - `MEMODATE` 가 GET 응답에서 `"2026-06-"` 처럼 잘려 오는 케이스 존재(`DateTime.tryParse` 실패 → now() fallback). 별도 확인 권장.

## 8. 학습 포인트
- 핵심 개념: **서버 mutation(생성/수정) 응답이 "영향 행 수 + 새 식별자"만 반환하는 패턴**(MySQL OkPacket 류)에서는,
  클라이언트가 응답을 곧바로 표시 모델로 쓰면 안 되고 **요청 입력값 + 응답의 식별자**로 모델을 재구성해야 한다.
  (또는 생성 후 단건 GET 으로 정본을 다시 받아오기.)
- 이번에 배운 점: Mock 이 "정상"이라고 실서버도 정상인 게 아니다 — Mock 은 입력값으로 풍부한 DTO 를 돌려주지만
  실서버는 빈약한 응답을 줄 수 있어, **응답 스키마 차이가 Mock 에 가려진다.**
- 다음에 같은 증상이면 확인할 순서:
  1) POST/PUT 응답 body 원문 확인 — 본문 필드가 실제로 들어오는가?
  2) 응답을 모델로 파싱하는 코드가 빈 키를 기본값으로 덮는가?
  3) Mock 과 실서버 응답 스키마가 다른가?
- 👉 별도 학습 노트: [[2026-06-06-mutation-response-identifier-only]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 create/update 서비스 작성 시 "응답에 표시용 필드가 다 들어오는가" 확인. 없으면 입력값으로 재구성 or 재조회.
  - [ ] Mock 응답을 실서버 응답 스키마와 **동일하게(빈약하게)** 맞추거나, 최소한 차이를 directive 에 명시.
- 추가하면 좋을 테스트 / 가드 / 로깅:
  - create 결과 DTO 의 `memo` 가 비어있지 않은지 단위 테스트(실서버 응답 픽스처로).
  - 응답 파싱 시 필수 필드 누락을 debugPrint 로 경고.
- 자동화 후보: Mock/실서버 응답 스키마 계약 테스트(contract test).

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: 저장은 성공(200)인데 방금 추가/수정한 항목이 화면에 **빈 값/일부 필드 누락**으로 보이고, 새로고침하면 정상.
- 확인 절차:
  1) 해당 mutation 의 POST/PUT 응답 body 를 로그로 확인.
  2) `data` 안에 표시에 필요한 필드(MEMO 등)가 실제로 있는지 본다.
  3) 응답→모델 파싱부에서 빈 키가 기본값으로 채워지는지 확인.
- 조치 절차: 응답의 식별자만 취하고 나머지는 요청 입력값으로 DTO 구성. 작성자 등 입력에 없는 값은 호출측에서 주입.
- 정상 확인 기준: 저장 직후(새로고침 없이) 입력 내용·날짜·고정·작성자명이 그대로 카드에 표시됨.

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "상요구 작성하기를 누르고 나서 아 아니 새로운 주소증을 작성하고 나서 화면에 ㅇ시 빈 값으로 나오게 되는데 확인해줄래? 그리고 내가 어떻게 고치면 되는지 step by step 으로"
- "이게 내가 코드 수정이 처음인데 return 으로 Dtro 를 넘기게 됫을 때는 어디서 이 코드를 받아서 ui 로 구성하는건지 궁금해 보다 자세히 step by step 으로 알려줄래?"
- "또한 emplName은 빈 값으로 넣는다는데 무슨말인지 모르겠네"
- "Jusojung 객체에서 savedRecordValues는 어디서 오는거야"
- "사용자가 입력값을 저장하는 부분은 어디고 ? 거기서 buildRecord를 호출하는건가?"
- "아래 코드에서 create 부분에 _service는 patientMemoServiceProvider 알겠는데 이거의 Dtio? 구조는 어떻게 보는지 궁금하네"

### Claude가 되물은 확인 질문 + 답
- Q: (이번 세션에 별도 확인 질문 없이 로그·코드 근거로 원인 확정) / A: 해당 없음

## 12. 한 줄 결론
> 서버의 생성/수정 응답이 식별자(insertId/MEMOID)만 줄 때는, 응답을 표시 모델로 직접 쓰지 말고 **입력값 + 응답 식별자**로 DTO 를 재구성해야 한다.
