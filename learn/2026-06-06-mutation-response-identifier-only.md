# 생성/수정 응답이 "식별자만" 줄 때 — 클라이언트 모델 재구성

> 한 줄 요약: 서버의 create/update 응답이 새 ID(insertId 등)와 영향 행 수만 반환하면, 그 응답을 표시 모델로 직접 쓰지 말고 **요청 입력값 + 응답의 식별자**로 모델을 재구성하거나 단건 재조회한다.

## 핵심
- 많은 백엔드가 INSERT/UPDATE 응답으로 **본문 전체가 아니라 결과 메타만** 돌려준다.
  (MySQL OkPacket: `affectedRows`, `insertId`, `warningStatus` … + 새 PK)
- 이 응답을 그대로 `Model.fromJson` 하면, 본문 필드(내용/날짜/작성자 등)는 키가 없어
  **기본값(빈 문자열·0)** 으로 채워진다 → 화면에 "방금 저장한 게 빈 값"으로 보임.
- 해결은 둘 중 하나:
  1) **입력값으로 재구성** — 응답에서는 새 ID 만 취하고, 나머지 필드는 방금 보낸 요청 값으로 모델을 만든다.
     (입력에 없는 값(작성자명 등)은 호출측에서 주입.)
  2) **재조회(read-after-write)** — 생성 후 단건 GET 으로 서버 정본을 다시 받아 표시.
- 낙관적 UI 와 잘 맞는 건 (1). 서버 계산 필드(파생값/포맷)가 중요하면 (2) 가 안전.

## 언제 / 왜 쓰나
- REST API 의 POST/PUT 응답이 `{insertId, affectedRows}` 류일 때.
- 저장 직후 목록/상세에 **새로고침 없이** 항목을 반영해야 할 때(자동저장·낙관적 추가).
- Mock 은 정상인데 실서버에서만 빈 값이 나올 때 — Mock 이 입력값으로 풍부한 응답을 흉내 내면
  실서버의 빈약한 응답 스키마가 가려진다. **Mock 을 실서버 응답과 같은 형태로** 맞추면 조기 발견된다.

## 예시 (코드·명령)
```dart
// 서버 POST 응답: {"data":{"insertId":18016,"affectedRows":1,"MEMOID":18016}}  // 본문 필드 없음

// ❌ 빈 값이 화면까지 새는 패턴
return Model.fromJson(res.data['data']); // memo:'', date:'', author:'' …

// ✅ 응답의 식별자 + 요청 입력값으로 재구성
final newId = (data['MEMOID'] as num?)?.toInt()
           ?? (data['insertId'] as num?)?.toInt() ?? 0;
return Model(
  id: newId,
  memo: memo, memoDate: memoDate, top: top,   // 요청에 보낸 입력값
  author: authorName,                          // 입력에 없으면 호출측에서 주입
);
```

## 흔한 함정 / 헷갈리는 점
- `fromJson` 이 누락 키를 **조용히 기본값으로** 채워서 버그가 에러 없이 지나간다(파싱은 성공).
- create 만 고치고 update 를 빠뜨림 — 보통 PUT 응답도 같은 형태라 같이 고쳐야 한다.
- Mock 이 "정상"이라 안심하기 쉬움 → 실서버 응답 스키마를 반드시 직접 확인.
- 재구성 시 입력에 없는 필드(작성자·서버 타임스탬프)는 비게 된다 → 호출측 주입 or read-after-write 로 보완.

## 질문 기록
### 내가 던진 질문 (원문)
- "또한 emplName은 빈 값으로 넣는다는데 무슨말인지 모르겠네"

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 기록할 확인 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-06-jusojung-create-empty-value]]
- 참고 링크: (없음)

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
