# 클라이언트 무결성 검사의 한계 & BuildConfig.DEBUG 우회의 안전성

> 한 줄 요약: APK 안에 든 무결성 검사(서명 해시 비교)는 리패키징 가능한 공격자에겐 통째로 제거 가능한 "성가심 수준" 방어다. `if (BuildConfig.DEBUG) skip` 분기는 release를 약화시키지 않는다(컴파일 상수 false, 바꾸려면 재서명 필요). 진짜 방어는 서버 검증/Play Integrity, 그리고 "debug 빌드를 배포하지 않는다".

## 핵심
- **release는 이 우회로 약해지지 않는다:** `BuildConfig.DEBUG`는 런타임 플래그가 아니라 **컴파일 시점 상수**. release APK엔 `false`가 박히고, 그 분기는 죽은 코드(R8이 제거 가능). `true`로 만들려면 APK를 수정→재컴파일→**재서명**해야 하는데, 재서명하면 서명 해시가 바뀌어 어차피 무결성에 걸린다.
- **근본 한계:** `EXPECTED_SIGNATURE_HASH`도 검사 로직도 전부 클라이언트(APK) 안에 있다. APK를 리패키징할 수 있는 공격자는 `verifySignature`를 `return true`로 패치하는 등 **검사 자체를 무력화**할 수 있다. debug 우회를 넣든 안 넣든 마찬가지 → 클라이언트 검사는 **defense-in-depth(성가심)**이지 철벽이 아니다.
- **이 우회가 만드는 실제 새 위험:** debug 서명 APK가 무결성 없이 돈다. 하지만 debug 빌드는 원래 `FLAG_DEBUGGABLE=true`라 chrome://inspect로 내부를 들여다보고 조작 가능 → **이미 그 자체로 보안이 없는 빌드**. 그래서 추가 위험은 "유출 시" 한정이고, 대응은 운영 규칙: **debug 빌드는 외부 배포·스토어 업로드 금지.**
- **진짜 방어선:** ① 민감 요청마다 **서버 측 검증**(앱 해시/버전) ② **Play Integrity API**(구 SafetyNet) ③ release JS 난독화(이미 build.sh가 함).

## 언제 / 왜 쓰나
- "무결성 검사를 우회하는 코드 넣어도 되나?" 안전성 판단할 때.
- 클라이언트 측 위변조 방어의 기대치 설정(철벽 X, 비용 상승 O).

## 예시 (코드·명령)
```java
// 컴파일 상수 — release에서는 false로 박힘
result.put("isDebug", BuildConfig.DEBUG);
```
```javascript
// debug에서만 검사 스킵; release(isDebug=false)는 기존 검증 유지
if (info.isDebug) { onSuccess({ skipped: true, reason: 'debug_build' }); return; }
```

## 흔한 함정 / 헷갈리는 점
- "클라이언트에 해시 박아두면 안전" → ❌. 박힌 값/로직은 추출·패치 가능.
- "debug 우회 = release 구멍" → ❌. `BuildConfig.DEBUG`는 빌드 시 분리. release엔 영향 없음.
- "무결성 통과 = 정품 보장" → ❌. 통과는 "쉽게 변조 안 됨" 정도. 보장은 서버/Play Integrity 몫.
- **운영 규칙이 핵심:** debug(AAB/APK)를 절대 스토어/외부로 내보내지 말 것.

## 강화 옵션(선택, 미적용)
- debug에서도 **서버 검사는 호출**하되(통계/로그 유지) 실패해도 종료만 안 하게.
- `BuildConfig.DEBUG` 대신 **debug 키 해시 일치**까지 요구(아무 debug 키나 통과 못 하게).

## 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "근데 이게 해킬 위험은 없나"

### Claude가 되물은 확인 질문 + 답
- Q: 지금 버전(BuildConfig.DEBUG 스킵)으로 빌드할까요, 아니면 강화 옵션을 반영할까요? / A: "일단 커밋" (현재 버전 유지 후 커밋)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-native-integrity-check]]
- 연결 노트: [[2026-06-02-android-app-signing-cert-hash]], [[2026-06-02-chrome-inspect-webview-debugging]]
- 참고 링크: Google Play Integrity API

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02
