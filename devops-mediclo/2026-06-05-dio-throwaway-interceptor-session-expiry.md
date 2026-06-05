# 2026-06-05 · 서비스마다 새로 만든 Dio가 인터셉터를 잃어 401 세션만료 자동 처리가 조용히 안 되던 구조적 결함

## 1. 개요
- 날짜: 2026-06-05
- 프로젝트 / 화면 / 모듈: devops-mediclo / 네트워크 레이어(`lib/core/network`) + 4개 API 서비스(현황판·진료실 증상경과·상용구·환자검색)
- 이슈 유형: 구조적 결함(잠재 버그) / API 연동
- 계층: **API·네트워크** (클라이언트 네트워크 설정)
- 최종 상태: **해결** (commit `6d71a29`)
- 중요도: **높음** (인증 세션 만료가 특정 화면에서 자동 처리되지 않음)

> 성격 주의: 이 결함은 운영 중 크래시로 "관찰"된 게 아니라, 중복 코드 리팩토링(directive 062) 중 **코드를 읽다가 정적으로 발견**한 잠재 결함이다. 즉 "에러 로그를 보고 고친" 사건이 아니라 "구조상 반드시 깨지는 지점을 찾아 예방 수정한" 사건이다. (원칙 1: 없는 재현 로그를 지어내지 않음)

## 2. 환경
- OS / 기기: (해당 없음 — 코드 구조 문제, 특정 기기 무관)
- 앱 버전 / 빌드: develop 브랜치 작업분
- 브랜치 / 커밋: `develop`, 수정 커밋 `6d71a29`
- 관련 의존성: `dio`, `flutter_riverpod`

## 3. 증상
### 관찰된 현상
- 표면적으로는 **아무 증상이 없었다.** 4개 서비스는 평소엔 정상 동작했다(토큰을 손으로 붙여놨기 때문).
- 그러나 구조상, 이 4개 서비스가 호출하는 엔드포인트에서 **토큰이 만료되어 서버가 401/403 을 돌려줘도 "세션 만료 → 로그인 화면 리디렉션" 자동 처리가 발동하지 않는다.** 사용자는 그냥 "조회 실패" 같은 일반 에러만 보고, 왜 실패했는지(세션 끊김) 모른 채 멈춘다.
- 부수적으로 이 경로에선 **요청/응답 로깅**과 **공통 에러 변환**도 빠져 있었다.

### 재현 절차 (구조상 예상되는 시나리오)
1. 로그인해서 토큰을 받는다.
2. 시간이 지나 토큰이 서버에서 만료된다.
3. 현황판/증상경과/상용구/환자검색 중 하나를 연다 → 해당 서비스가 401 을 받는다.
4. **기대**: 세션 만료 감지 → 전역 `sessionExpiredProvider` 켜짐 → 로그인 화면으로 보냄.
   **실제(수정 전)**: 아무 일도 안 일어나고 그냥 일반 실패로 끝남.

> ⚠️ 위 4번은 코드 경로 분석으로 도출한 결과다. 실제 만료 토큰으로 재현 측정까지 하진 않았다.

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
(해당 없음 — 런타임 에러가 아니라 "있어야 할 동작이 조용히 빠진" 구조적 누락)
```

## 4. 확인한 내용
- `grep -rn "_getDio\|_authHeader"` 로 동일 패턴이 4개 서비스에 **바이트 단위로 복제**돼 있음을 확인.
- `lib/core/network/api_client.dart` 의 `dioProvider` 가 어떤 인터셉터를 다는지 확인 → Log / Error / **Auth**(세션만료 콜백 포함).
- `lib/core/network/auth_interceptor.dart` 의 `AuthInterceptor.onError` 가 `statusCode == 401 || 403` 일 때 `_handleSessionExpired()` 를 호출함을 확인 (이게 세션 만료 처리의 핵심).
- 각 서비스의 `_getDio()` 가 `dioProvider` 가 아니라 **그 자리에서 `Dio(BaseOptions(...))` 를 새로 생성**해 반환함을 확인 → 이 새 Dio 에는 인터셉터가 하나도 안 붙는다.
- 결론: 그래서 그 새 Dio 로 나간 요청의 401 응답은 `AuthInterceptor` 를 거치지 않으므로 세션만료 처리가 발동할 수 없다.

## 5. 원인 분석

### 확정 원인
**"baseUrl 을 바꾸려고 서비스가 매 요청마다 인터셉터 없는 새 Dio 를 만들어 쓴 것"** 이 근본 원인이다.

배경부터 보면 — 이 앱은 로그인 후 서버가 알려준 기관별 주소(`serverUrl`)로 API 를 호출해야 한다. 그런데 앱 시작 시 만들어 둔 공용 Dio(`dioProvider`)의 baseUrl 은 고정값(`ApiConfig.baseUrl`)이라 기관 주소를 못 쓴다. 그래서 각 서비스가 이런 우회 코드를 복붙해서 썼다:

```dart
// ❌ 문제의 코드 형태 (4개 서비스에 동일 복제)
Dio _getDio() {
  final serverUrl = _authStorage.getServerUrl();
  if (serverUrl != null && serverUrl.isNotEmpty) {
    return Dio(                       // ← 여기! 완전히 새 Dio 객체
      BaseOptions(baseUrl: '$serverUrl/v2', /* timeouts */),
    );
    // 이 새 Dio 의 interceptors 리스트는 [] (비어 있음).
    // dioProvider 에 달아둔 Log/Error/Auth 인터셉터가 전부 안 따라온다.
  }
  return _dio;
}

Map<String, String> _authHeader() {
  final token = _authStorage.getToken();
  if (token != null && token.isNotEmpty) {
    return {'Authorization': token, 'Content-Type': 'application/json'};
  }
  return {'Content-Type': 'application/json'};
}

// 호출부
final dio = _getDio();
final res = await dio.get('/path', options: Options(headers: _authHeader()));
```

여기서 핵심은 **`Dio(...)` 는 "설정을 바꾼 같은 Dio" 가 아니라 "완전히 다른 새 객체"** 라는 점이다. Flutter 를 처음 하면 헷갈리기 쉬운 부분인데:

- `dioProvider` 가 만든 Dio: `dio.interceptors` 에 Log·Error·Auth 가 **들어 있음**.
- `_getDio()` 가 만든 Dio: 방금 `new` 한 빈 객체라 `dio.interceptors` 가 **텅 비어 있음**.

인터셉터(interceptor)는 "요청이 나가기 직전/응답이 들어온 직후에 거치는 검문소" 다. `AuthInterceptor` 라는 검문소가 ① 나갈 때 토큰을 붙이고 ② 들어올 때 401 이면 세션만료 처리를 한다. 그런데 새 Dio 엔 이 검문소가 없으니 둘 다 안 일어난다.

그러면 토큰은 어떻게 붙었나? → **`_authHeader()` 가 손으로 붙여서** 다. 이게 함정이다. `_authHeader()` 가 ①(토큰 부착)은 흉내 내줘서 **평소엔 멀쩡히 동작**하지만, ②(401 세션만료 감지)는 아무도 대신 안 해준다. 즉 `_authHeader()` 는 진짜 문제(인터셉터 통째 누락)를 가려버린 **반창고(band-aid)** 였다. 그래서 버그가 "조용히" 숨어 있었다.

### 가능성 있는 원인 (기여 요인)
- 처음 한 서비스(`status_board_service`)가 이 패턴을 썼고, 이후 서비스들이 "이 패턴 그대로 복제"라고 주석까지 달며 복붙됨 → 결함이 4곳으로 전파.

## 6. 조치 내역

### 근본 수정 (root fix)
"baseUrl 만 바꾸고 싶은데 인터셉터는 유지" 를 정공법으로 해결: **baseUrl 교체도 인터셉터로 만든다.**

```dart
// ✅ 신규: 요청마다 baseUrl 을 동적으로 갈아끼우는 인터셉터
class DynamicBaseUrlInterceptor extends Interceptor {
  DynamicBaseUrlInterceptor(this._authStorage);
  final AuthStorage _authStorage;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final serverUrl = _authStorage.getServerUrl();
    if (serverUrl != null && serverUrl.isNotEmpty) {
      options.baseUrl = '$serverUrl/v2';   // 새 Dio 안 만들고, 이번 요청 설정만 수정
    }
    handler.next(options);
  }
}
```

그리고 이 인터셉터 + 기존 Log/Error/Auth 를 **모두** 붙인 Dio 를 Provider 로 제공:

```dart
final v2DioProvider = Provider<Dio>((ref) {
  final authStorage = ref.watch(authStorageProvider);
  return _buildV2Dio(                       // Log + Error + Auth + DynamicBaseUrl
    authStorage: authStorage,
    onSessionExpired: () =>
        ref.read(sessionExpiredProvider.notifier).state = true,
  );
});
```

서비스는 이제 이 Dio 만 주입받아 **그냥 `_dio.get('/path')`** 만 한다. `_getDio()`/`_authHeader()` 는 통째로 삭제.

```dart
// 수정 후 서비스 (예: hot_key_tag_service)
HotKeyTagServiceImpl(this._dio);          // v2DioProvider 주입
final res = await _dio.get('/consultation/tags');   // 토큰·baseUrl·401처리 전부 인터셉터가 담당
```

추가로 4벌 중복이던 `_handleDioError` 도 제네릭 헬퍼 `mapDioError<F>()` 로 합침.

### 조치 결과
- 4개 서비스에서 `_getDio`/`_authHeader` grep **0건**.
- `flutter analyze` error 0건, 관련 단위 테스트(status_board 26 + patient_search 추가) **전부 통과**.
- 삭제된 중복 약 96줄.
- **세션 만료(401) 자동 리디렉션 경로 복구** (이 4개 서비스도 이제 `AuthInterceptor` 를 거침).

> 범위 메모: `auth_service`(로그인 부트스트랩)·`consent_service`(패턴 미사용)·모든 Mock 서비스는 손대지 않음. `hot_key_tag_service` 는 아직 미커밋(061 작업)이라 062 커밋에는 3개 서비스만 포함됨.

## 7. 최종 상태
- 해결 여부: **해결** (commit `6d71a29` on `develop`)
- 남은 문제: `hot_key_tag_service.dart` 등 061 미커밋 파일은 별도 커밋에서 v2dio 적용분이 함께 들어갈 예정.

## 8. 학습 포인트
- 핵심 개념:
  1. **`Dio(...)` 는 "설정 바꾼 같은 Dio" 가 아니라 "인터셉터 0개짜리 새 객체"** 다. 새로 만들면 공용 인터셉터를 전부 잃는다.
  2. **인터셉터 = 횡단 관심사(인증·로깅·세션만료)의 단일 지점.** 서비스가 손으로 흉내 내면 *일부만* 흉내 나고 나머지(세션만료 등)는 조용히 빠진다.
  3. **반창고가 진짜 버그를 숨긴다** — `_authHeader()` 가 토큰을 손으로 붙여줘서 "평소엔 됨" 이 됐고, 그게 401 처리 누락을 가렸다.
- 다음에 같은 증상이면 확인할 순서:
  1) 그 서비스가 쓰는 Dio 가 `dioProvider`/`v2DioProvider`(인터셉터 있음)인지, 아니면 `Dio(...)`(빈 객체)인지 확인.
  2) `Options(headers: ...)` 로 토큰을 손으로 붙이고 있으면 → 인터셉터를 우회 중이라는 신호. 의심.
  3) 401/403 응답이 세션만료 처리로 이어지는지 `AuthInterceptor.onError` 경로를 따라가 본다.
- 👉 별도 학습 노트: [[2026-06-05-dio-interceptor-auth-delegation]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 API 서비스는 **반드시 `v2DioProvider`(또는 `dioProvider`)를 주입**받는다. 서비스 안에서 `Dio(...)` 를 직접 생성하지 않는다.
  - [ ] `Options(headers: {'Authorization': ...})` 를 서비스 코드에서 손으로 쓰지 않는다(인터셉터가 한다).
  - [ ] baseUrl 을 바꿔야 하면 새 Dio 가 아니라 **인터셉터**로 바꾼다.
- 추가하면 좋을 가드:
  - lint/grep CI: `lib/features/**/services/*.dart` 안에서 `Dio(` 또는 `_authHeader` 사용 시 경고.
  - 위젯/통합 테스트로 "401 응답 → sessionExpiredProvider 가 true 가 되는가" 를 한 번 검증.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: "특정 화면만 토큰 만료돼도 로그인 화면으로 안 튕기고 그냥 실패함" / "특정 서비스 요청만 네트워크 로그에 안 찍힘".
- 확인 절차:
  1. 그 화면이 쓰는 service 파일을 열어 주입받는 Dio 가 무엇인지 본다.
  2. `Dio(BaseOptions(...))` 가 보이면 → 인터셉터 없는 깡통 Dio. 원인 확정에 가깝다.
  3. `_authHeader()` 같은 수동 헤더가 있으면 → 인터셉터 우회 흔적.
- 조치 절차: `v2DioProvider` 주입으로 바꾸고, `_getDio()`/`_authHeader()` 삭제, baseUrl 은 `DynamicBaseUrlInterceptor` 에 맡긴다.
- 정상 확인 기준: 만료 토큰(또는 모킹된 401)으로 그 화면을 열었을 때 로그인 화면으로 리디렉션되고, 요청이 네트워크 로그에 찍힌다.

## 11. 질문 기록

### 내가 던진 질문 (원문)
- "아 백엔드 엔지니어에게 ... 이 부분이 services 모듈에 전체적으로 겹치는거 같은데 뭔가 좋은 방법 없는지 확ㅇ니해줘봐"
- "ㅇㅇ" (리팩토링 디스패치 승인)
- "커밋하고"
- "어떤 버그가 있었는지 문제가 무엇ㄴ이었는지 그 코드의 형태는 어떤 문제가 터지는지 flutter 처음 하는 사람의 수준으로 구조적인 문제에 대해서 보다 자세히 넣은 것을 만들어줄래?" (이 문서의 요청)

### Claude가 되물은 확인 질문 + 답
- Q: 중복 제거를 어느 수준까지 갈까요? (A: Mixin 추출 / B: v2DioProvider 근본 해결 / C: A+에러매퍼 / 분석만)
  - A: **"B - v2DioProvider 근본 해결"** 선택.

## 12. 한 줄 결론
> `Dio(...)` 로 새 객체를 만들면 인터셉터(인증·세션만료·로깅)가 통째로 빠지고, `_authHeader()` 같은 수동 반창고가 그 누락을 가려 "평소엔 되는데 만료 시 조용히 안 되는" 버그가 된다 — baseUrl 교체조차 인터셉터로 만들어 하나의 공용 Dio(`v2DioProvider`)에 위임하는 게 정답.
