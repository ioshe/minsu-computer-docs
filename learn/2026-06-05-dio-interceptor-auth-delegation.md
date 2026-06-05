# Dio 인증·baseUrl을 인터셉터에 위임하기 (Thin Service 패턴)

> 한 줄 요약: API 서비스마다 `_authHeader()` / `_getDio()` 를 직접 짜지 말고, **토큰 부착·baseUrl 재해석은 Dio 인터셉터에 한 번만 심고**, 서비스는 `_dio.get('/path')` 만 호출하는 얇은 코드로 둔다.

## 핵심

- Dio 는 요청이 나가기 전 `onRequest` 단계에서 **인터셉터 체인**을 통과한다. 여기에 `Authorization` 헤더 부착, baseUrl 교체 같은 **횡단 관심사(cross-cutting concern)** 를 심으면, 모든 서비스가 그 처리를 공짜로 얻는다.
- 따라서 인증 토큰을 서비스 코드가 매번 `Options(headers: _authHeader())` 로 손수 붙이는 건 **인터셉터가 이미 하는 일을 중복**으로 하는 것이다.
- 더 위험한 건, 서비스가 `_getDio()` 로 **인터셉터 없는 새 Dio 를 만들어 쓰는 경우**다. 이러면 공용 인터셉터(인증·세션만료·로깅)가 전부 빠진 "깡통 Dio" 가 되어, 빠진 만큼을 서비스가 손으로 다시 채워야 한다 → 누락·불일치의 온상.
- 올바른 방향: **인증/baseUrl 처리가 다 들어있는 Dio 를 주입(DI)** 받아 그대로 쓴다. 서비스는 엔드포인트와 직렬화만 책임진다 (단일 책임).

## 언제 / 왜 쓰나

- REST 서비스가 여러 개고, 모두 같은 토큰·같은 baseUrl 규칙을 따를 때.
- 토큰 만료·세션 리디렉션 같은 정책을 **한 곳에서** 갈아끼우고 싶을 때 (인터셉터만 고치면 전 서비스 반영).
- 새 API 서비스를 추가할 때 보일러플레이트를 0 으로 만들고 싶을 때.

## 이 프로젝트(mediclo)에서의 실제 구성

`lib/core/network/api_client.dart` 에 Dio Provider 가 두 개 있다.

| Provider | 인터셉터 구성 | baseUrl | 용도 |
|----------|--------------|---------|------|
| `dioProvider` | Log + Error + **Auth** | `ApiConfig.baseUrl` (정적) | 일반 |
| `v2DioProvider` | Log + Error + **Auth** + **DynamicBaseUrl** | `{serverUrl}/v2` (요청마다 동적) | v2 API 권장 |

- `AuthInterceptor.onRequest` 가 `options.headers['Authorization'] = token` 을 자동 부착 (`/login` 만 예외). → `auth_interceptor.dart:42`
- `DynamicBaseUrlInterceptor.onRequest` 가 요청마다 `options.baseUrl = '$serverUrl/v2'` 로 교체. → `dynamic_base_url_interceptor.dart:25`
- 그래서 `v2DioProvider` 를 주입받은 서비스는 `_getDio()` 도 `_authHeader()` 도 필요 없다. (api_client.dart:77 주석에 명시되어 있음)

## 세 가지 패턴 비교 (이 레포에 셋 다 공존)

```
패턴 A) 두꺼운 서비스 — _authHeader() + _getDio()  ← 레거시/중복
  hot_key_tag_service.dart, consultation_progress_service.dart

  fetchTags()
    └ _getDio()  → ⚠️ 인터셉터 0개짜리 새 Dio 생성
        └ Options(headers: _authHeader())  → 토큰 손수 부착
            └ dio.get('/consultation/tags')
                └ status/code/String·Map 분기까지 서비스가 전부 처리

패턴 B) 얇은 서비스 — dioProvider 주입  ← real_patient_memo_service.dart
  RealPatientMemoService(this._dio)   // dioProvider 주입
    └ _dio.get/post/put/delete(...)   // 끝. _authHeader 없음
        └ [AuthInterceptor 가 토큰 자동 부착]
        └ baseUrl 은 정적(ApiConfig.baseUrl) — 동적 serverUrl 은 못 씀

패턴 C) 가장 올바름 — v2DioProvider 주입  ← 권장
    └ _dio.get('/path')
        └ [AuthInterceptor]        토큰 자동
        └ [DynamicBaseUrlInterceptor]  {serverUrl}/v2 자동
        // 인증·baseUrl 둘 다 위임. 서비스는 순수 endpoint 만.
```

### 요청 1건이 흐르는 플로우

```
[서비스] _dio.get('/consultation/patient-memo', queryParameters: {...})
   │
   ▼
┌──────────────── Dio 인터셉터 체인 (onRequest) ────────────────┐
│ DynamicBaseUrlInterceptor :  options.baseUrl = {serverUrl}/v2 │  ← baseUrl 주입
│ LogInterceptor            :  요청 로깅                         │
│ _ErrorInterceptor         :  (onError 대비)                    │
│ AuthInterceptor           :  headers['Authorization'] = token │  ← 토큰 주입
└──────────────────────────────┬───────────────────────────────┘
                               ▼
                    실제 HTTP 요청 전송  →  서버
                               │
                               ▼ (onResponse)
              AuthInterceptor : 세션 만료면 reject → 전역 리디렉션
                               ▼
                    [서비스] res.data 파싱 (DTO.fromJson)
```

> 패턴 A 가 위험한 이유가 이 그림에서 명확해진다: `_getDio()` 가 만든 새 Dio 에는 **이 체인 자체가 없다.** 그래서 토큰을 손으로 붙여야 하고, 세션 만료 처리·로깅·에러 변환은 통째로 누락된다. 한 군데서 토큰 형식이 바뀌면 패턴 A 서비스들만 조용히 깨진다.

## 왜 패턴 B/C 가 "올바른" 방식인가 (정리)

1. **DRY** — 토큰·baseUrl 로직이 인터셉터 한 곳에만. A 는 서비스 수만큼 복붙됨.
2. **단일 책임** — 서비스는 "어떤 endpoint 를 어떤 DTO 로" 만 안다. "어떻게 인증하나" 는 네트워크 레이어 책임.
3. **정책 일원화** — 세션 만료 리디렉션, 토큰 갱신, 로깅을 인터셉터에서 한 번에 갈아끼움. A 는 누락되거나 제각각.
4. **테스트 용이** — 서비스 테스트는 Dio 만 mock 하면 됨. A 는 AuthStorage 까지 끌고 와야 함.
5. **누락 안전** — 새 서비스가 토큰 붙이는 걸 깜빡할 수 없음(인터셉터가 강제). A 는 깜빡 가능.

## 흔한 함정 / 헷갈리는 점

- **"얇게 만들려고 `_authHeader()` 만 지웠다"** → 그 서비스가 `_getDio()` 로 인터셉터 없는 Dio 를 쓰고 있었다면 **토큰이 통째로 빠진다.** 반드시 인터셉터가 붙은 Dio(`v2DioProvider`)를 주입하는 것까지 같이 바꿔야 한다.
- `dioProvider`(패턴 B) 는 인증은 자동이지만 **baseUrl 은 정적**이다. 로그인 시 기관별 serverUrl 이 동적으로 바뀌는 v2 엔드포인트라면 `v2DioProvider`(패턴 C)를 써야 한다.
- 인터셉터 **등록 순서**가 동작 순서다. baseUrl 교체와 인증 부착이 서로 의존하지 않으면 순서는 무관하지만, 의존하면(예: baseUrl 따라 토큰이 달라짐) 순서를 의식해야 한다.
- `/login` 처럼 토큰을 붙이면 안 되는 엔드포인트는 인터셉터 안에서 예외 처리한다 (`auth_interceptor.dart:36`). 서비스가 분기하지 않는다.
- Either(Failure) vs throw — 인증/baseUrl 위임과는 **별개 축**이다. 얇은 서비스라도 에러 매핑은 공용 `mapDioError`(api_client.dart:150) 로 통일할 수 있다.

## 질문 기록
> 이 항목은 항상 채운다(원칙 8).

### 내가 던진 질문 (원문)
- "patient memo  service 에서는   Map<String, String> _authHeader() { 해당 부분이 어디있는거야?"
- "그럼 다른 services 랑 뭐가 다른거야 ?"
- "플로우 다이어그램 만들어서 왜 올바른 방식인지 정리" (이 노트의 요청)

### Claude가 되물은 확인 질문 + 답
- Q: hot_key_tag / consultation_progress 도 patient_memo 처럼 인터셉터 위임 방식으로 리팩토링할까요? / A: ❓ 미답 (대신 "왜 올바른지"를 문서로 정리하라는 지시로 이어짐)

## 관련
- 발단이 된 이슈: (별도 이슈 없음 — 코드 비교/이해 과정에서 파생)
- 참고 코드:
  - `lib/core/network/api_client.dart` (`dioProvider`, `v2DioProvider`, `_buildV2Dio`)
  - `lib/core/network/auth_interceptor.dart` (`AuthInterceptor`)
  - `lib/core/network/dynamic_base_url_interceptor.dart`
  - 패턴 B 예시: `lib/features/consultation/data/services/real_patient_memo_service.dart`
  - 패턴 A 예시: `lib/features/consultation/data/services/hot_key_tag_service.dart`, `consultation_progress_service.dart`

---
- 생성일: 2026-06-05
- 마지막 갱신: 2026-06-05
