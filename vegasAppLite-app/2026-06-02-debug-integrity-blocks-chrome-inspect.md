# 2026-06-02 · debug 빌드 무결성 검사가 chrome://inspect 디버깅을 막음 → debug 우회

## 1. 개요
- 날짜: 2026-06-02
- 프로젝트 / 화면 / 모듈: vegasAppLite-app / cordova-plugin-integrity (앱 무결성 검사)
- 이슈 유형: 버그(설계상 부작용) / 디버깅 환경
- 계층: 네이티브·플러그인 (Cordova 무결성 플러그인 + 서명) + 클라이언트(JS)
- 최종 상태: 해결 (커밋 f9c7101, 빌드 검증 전)
- 중요도: 높음

## 2. 환경
- OS / 기기 / 브라우저·웹뷰: Linux 데스크톱 + 안드로이드 실기기, System WebView(Chromium)
- 앱 버전 / 빌드: 1.1.29 (versionCode 56), applicationId `com.trustnhope.vegas`, debug 빌드
- 브랜치 / 커밋: feature/isms / 수정 커밋 `f9c7101`
- 관련 의존성·버전: cordova-android (CordovaLib), build-tools 35.0.0, AGP 8.6.0, JDK 17

## 3. 증상
### 관찰된 현상
- **release 빌드는 정상** 동작/설치되는데, **debug 빌드는 앱 무결성 검증에 실패**.
- 본래 목적은 `chrome://inspect`로 WebView를 디버깅하는 것인데, debug 빌드가 무결성 실패로 **앱이 스스로 종료**되어 디버깅 불가.

### 재현 절차
1. `./build.sh debug` 로 debug APK 빌드
2. 기기에 설치 후 실행
3. `deviceready` 시점에 무결성 검사 실행 → 로컬 검사 실패 → `alert` 후 `navigator.app.exitApp()` 로 앱 종료
4. `chrome://inspect` 에 잠깐 떴다 사라져 디버깅 불가

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
(앱 내 alert)
앱 무결성 검증에 실패했습니다. 앱을 종료합니다.
```
```text
# 무결성 검사가 비교하는 값
EXPECTED_SIGNATURE_HASH = 8B717936301458296D0DE522B4C327F15C9AC53C66E478CA84398EB54C42F58B  (release 키)
debug APK 실제 서명(SHA-256) = c13b3a8d8571c5789b447e6bd1b1ec65d258ab4f003a19cdf1e19f7f0fb00edf  (debug 키)
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점:
  - `apksigner verify --print-certs <debug.apk>` → V2 cert SHA-256 = `c13b3a8d...` (debug 키)
  - `keytool -list -v -keystore keyStore/vegassolution_release_key.jks` → SHA256 = `8B7179...` (= 코드의 EXPECTED 해시와 일치)
  - `keytool -list -v -keystore ~/.android/debug.keystore` → SHA256 = `c13b3a8d...` (debug APK와 일치)
  - 서명 properties: `platforms/android/release-signing.properties` → `storeFile=../../keyStore/vegassolution_release_key.jks` (존재 확인, 경로 정상). release 서명 연결은 `platforms/android/app/build.gradle`이 `../release-signing.properties` 자동 감지(L119-121).
  - 무결성 비교: `IntegrityChecker.java:17`(EXPECTED 상수), `:91`(`EXPECTED_SIGNATURE_HASH.equals(currentHash)`).
  - 종료 흐름: `www/js/integrity.js` — `deviceready`(L164) → `verifyOnLaunch`(L128) → onFailure에서 `alert`(L137) + `navigator.app.exitApp()`(L139).
  - chrome://inspect 조건: `CordovaLib/.../engine/SystemWebViewEngine.java:176-178` — `FLAG_DEBUGGABLE`일 때만 `enableRemoteDebugging()` → `setWebContentsDebuggingEnabled(true)`(L226).
- 결과 해석: 서명 properties는 정상. 문제는 **무결성 검사가 release 인증서 해시만 통과시키도록 하드코딩**돼 있어, debug 키로 서명된 debug 빌드는 영원히 실패한다는 것.

## 5. 원인 분석
- **확정 원인:** 무결성 검사([IntegrityChecker.java:91])가 앱 서명 인증서의 SHA-256을 **하드코딩된 release 키 해시(8B7179…)**와만 비교한다. debug 빌드는 안드로이드 기본 debug 키(`~/.android/debug.keystore`, 해시 c13b3a8d…)로 자동 서명되므로 항상 불일치 → `false` → JS가 그 false를 보고 `exitApp()`.
- **딜레마 구조:**
  - release: 무결성 통과 ✅ but `FLAG_DEBUGGABLE=false` → chrome://inspect 안 뜸
  - debug: `FLAG_DEBUGGABLE=true` → chrome://inspect 뜨는 조건 ✅ but 무결성 실패로 즉시 종료 ❌
- 보조 사실: 로컬 검사를 통과시켜도, 이어지는 서버 검사(`verifyWithServer` → `https://ims2.vegas-solution.com/internal/AppVersionCheck`)가 debug 해시를 등록값과 대조해 또 실패(`verified===false`)하므로 **로컬+서버 둘 다** 우회해야 함.
- (정정) "에러가 터지는" 게 아니라 `verifySignature`가 조용히 `false`를 반환하고, JS가 그 값으로 앱을 종료시키는 흐름.

## 6. 조치 내역
- 수행한 조치 (root fix) — 커밋 `f9c7101`, 2파일 +10줄:
  1. **네이티브**: `getIntegrityInfo` 응답에 빌드 타입 노출
     ```java
     // plugins/cordova-plugin-integrity/src/android/IntegrityPlugin.java (getIntegrityInfo)
     result.put("isValid", isValid);
     result.put("isDebug", BuildConfig.DEBUG);   // ← 추가
     result.put("platform", "android");
     ```
     - `BuildConfig.DEBUG`는 컴파일 상수(debug=true / release=false). `IntegrityPlugin`이 `package com.trustnhope.vegas`라 import 불필요. `build.gradle`에 `buildConfig true`(L189) + `namespace com.trustnhope.vegas`(L175) 있어 생성됨.
  2. **JS**: `verify()` 초입에서 debug면 로컬·서버 검사 모두 스킵
     ```javascript
     // www/js/integrity.js — getIntegrityInfo 콜백 맨 앞
     if (info.isDebug) {
         console.warn(TAG, 'Debug build detected - skipping integrity verification');
         onSuccess({ skipped: true, reason: 'debug_build' });
         return;
     }
     if (!info.isValid) { ... }   // (기존 release 로직)
     ```
  3. **build.sh**: 18번 줄 한글 쓰레기 문자(`ㄴㅇㄻㄴㅇㄹ`) 제거 (이미 반영, 현재 clean)
- 플러그인 복사 구조 주의: `build.sh`가 빌드 시 `plugins/cordova-plugin-integrity/src/android/`의 Java를 `platforms/android/app/src/main/java/com/trustnhope/vegas/`로 복사(L153-154). **원본 `plugins/`를 고쳐야** 다음 빌드에 반영됨.
- 조치 결과: 코드 반영·커밋 완료. **빌드/기기 검증은 아직** (다음 액션).

## 7. 최종 상태
- 해결 여부: 코드 수정·커밋 완료 (해결), **실제 빌드·chrome://inspect 동작 확인은 미수행**
- 남은 문제 / 추가 확인 필요:
  - `./build.sh debug` → `adb uninstall com.trustnhope.vegas` → 설치 → chrome://inspect 노출/디버깅 확인
  - (선택) 강화: debug에서도 서버 검사는 돌리되 종료만 막기 / `BuildConfig.DEBUG` 대신 debug 키 해시 일치까지 요구
  - push 미실행 (로컬 커밋만)

## 8. 학습 포인트
- 핵심 개념:
  1. **서명 인증서 해시** 기반 무결성 검사는 빌드 타입(키)에 종속 → debug는 구조적으로 통과 불가
  2. **chrome://inspect = FLAG_DEBUGGABLE(=debug 빌드)** 조건
  3. `BuildConfig.DEBUG`로 debug/release 분기하면 release 안전 유지하며 debug만 우회 가능
- 이번에 배운 점: "release는 되는데 debug만 안 됨" → 서명 키 차이(인증서 해시)부터 의심.
- 다음에 같은 증상이면 확인할 순서: 1) APK 실제 서명(`apksigner`) vs 코드의 기대 해시 2) 검사 실패 시 동작(exit?) 3) 빌드 타입 분기 여부
- 👉 별도 학습 노트: [[2026-06-02-android-app-signing-cert-hash]], [[2026-06-02-chrome-inspect-webview-debugging]], [[2026-06-02-cordova-js-java-bridge]], [[2026-06-02-android-context-info]], [[2026-06-02-client-integrity-check-limits]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 무결성 검사는 `release`에만 적용, `debug`는 `BuildConfig.DEBUG`로 스킵
  - [ ] debug 빌드 배포 금지 (debuggable + 무결성 미적용이라 위험)
- 추가하면 좋을 테스트 / 가드 / 로깅: 무결성 검사 실패 사유를 콘솔/서버 로그로 명확히 남기기(이미 `console.warn` 추가)
- 자동화 후보: CI에서 release만 무결성 자가검증 후 산출물 게이트

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: debug에서 "앱 무결성 검증에 실패했습니다. 앱을 종료합니다." alert 후 종료
- 확인 절차: `apksigner verify --print-certs <apk>` 해시 vs `IntegrityChecker.EXPECTED_SIGNATURE_HASH`
- 조치 절차: `info.isDebug` 스킵 분기 존재 확인 → 없으면 추가 → `./build.sh debug` 재빌드
- 정상 확인 기준: debug 앱이 종료되지 않고 chrome://inspect에 WebView 타깃으로 표시

## 11. 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "무결성 검증에 실패했는데 현재 프로젝트 구조에서 properties 가 잘 세팅되었는지 확인 좀"
- "내가 하고 싶은 것은 chrome:inspect 에서 debuging 하고 싶은건데 이거 뜨게 하는 기준이 뭔지 알려줘 그리고 현재 상태 스크립트 점검하고"
- "/build.sh 로 ... vegasapp-debug-20260602_140102.apk 를 만들었는데 ... 디버그로 빌드를 했는데 이게 앱 무결성 검증이 실패하거든? release 로 넣으면 잘되는데 debug 일 때 왜 안되는지 확인해줄래?"
- "네이티브 getIntegrityInfo에 isDebug: BuildConfig.DEBUG 필드 추가 라는게 뭐야 내가 수정할 수 없다는건가?? 현재 프로젝트에서 수정 불가??"
- "일단 우회 수정을 하자 어떻게 할꺼야?"
- "근데 이게 해킬 위험은 없나"
- "일단 커밋"

### Claude가 되물은 확인 질문 + 답
- Q: (우회 수정을) 바로 적용해드릴까요? / A: "일단 우회 수정을 하자" (승인)
- Q: 빌드까지 진행할까요? (컴파일 통과/APK 생성 확인) / A: ❓ 미답 (대신 "일단 커밋" 요청 → 빌드 전 커밋만)
- Q: 강화 옵션(서버 검사 유지·debug 키 해시 일치)을 반영할까요, 지금 버전으로 빌드할까요? / A: "일단 커밋" (현재 버전 유지)

## 12. 한 줄 결론
> 무결성 검사가 release 인증서 해시만 통과시키니 debug는 구조적으로 실패→자동종료였다. `BuildConfig.DEBUG` 분기로 debug만 검사를 스킵해 release 보안은 그대로 두고 chrome://inspect 디버깅을 살렸다.
