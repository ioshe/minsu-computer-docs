# 2026-06-02 · debug 빌드에서 앱 무결성 검사 실패 — JS만 고쳐서 안 됐고 네이티브 MainActivity가 진짜 원인

## 1. 개요
- 날짜: 2026-06-02
- 프로젝트 / 화면 / 모듈: vegasAppLite-app / 앱 시작(스플래시~첫 화면) / 무결성 검증
- 이슈 유형: 버그 (debug 빌드에서 의도치 않게 앱이 즉시 종료)
- 계층: 네이티브·플러그인 (Cordova `MainActivity`) — 한때 클라이언트(JS)로 오인
- 최종 상태: 해결
- 중요도: 높음 (debug 빌드로 `chrome://inspect` 디버깅 자체가 불가능했음)

## 2. 환경
- OS / 기기 / 웹뷰: Android, Samsung 실기기(adb serial `R54XA008HCF`), System WebView(Chromium)
- 앱 버전 / 빌드: 1.1.29 / debug 변형(`assembleDebug`, debug 키 서명)
- 브랜치 / 커밋: feature/isms / f9c7101 (이번 수정 전 기준)
- 관련 의존성: Cordova Android, 자체 플러그인 `cordova-plugin-integrity`, AGP/Gradle 8.9

## 3. 증상
### 관찰된 현상
- debug APK 설치 후 앱을 실행하면 **"앱 무결성 검증에 실패했습니다."** 메시지가 뜨고 앱이 곧바로 닫힘.
- 이전 세션에서 "debug 빌드면 무결성 검사를 건너뛴다"고 고쳤는데(커밋 f9c7101), 여전히 실패가 뜸.
- 앱이 즉시 종료되니 `chrome://inspect` 디버깅도 불가능(원래 이걸 하려고 스킵을 넣은 것).

### 재현 절차
1. `./build.sh debug` 로 debug APK 빌드
2. `adb install -r output/vegasapp-debug-*.apk`
3. 기기에서 앱 실행 → Toast 뜨고 즉시 종료

### 에러 메시지 / 로그 (원문)
```text
06-02 17:58:33.443  IntegrityChecker: Current signature hash: C13B3A8D8571C5789B447E6BD1B1EC65D258AB4F003A19CDF1E19F7F0FB00EDF
06-02 17:58:33.444  E MainActivity: Integrity check failed!
```
화면 Toast 문구: `앱 무결성 검증에 실패했습니다.`

## 4. 확인한 내용
시간순으로 좁혀 들어간 과정:

1. **이전 수정이 코드에 들어있나** — `git show f9c7101` 로 JS([www/js/integrity.js])와 플러그인 Java(`IntegrityPlugin.java`)에 `isDebug` 스킵이 들어간 걸 확인. 코드 자체는 맞음.
2. **빌드 산출물에 반영됐나** — `platforms/android/.../assets/www/js/integrity.js`(번들 JS)와 `platforms/.../IntegrityPlugin.java` 둘 다 `isDebug` 로직 존재 확인.
3. **설치된 APK 안까지 들어갔나** — `unzip -p <apk> assets/www/js/integrity.js | grep isDebug` 로 **APK 번들 JS에도** 스킵 로직 존재 확인. → JS 경로는 완벽한데도 실패.
4. **그럼 런타임에서 isDebug가 false인가?** 를 보려고 logcat 확인 → 여기서 결정적 단서 발견:
   - `MainActivity: Integrity check failed!` 라는 **네이티브** 로그가 찍힘.
   - 즉 실패를 내는 주체가 JS(`integrity.js`)가 아니라 **네이티브 `MainActivity`** 였음.
5. `grep`으로 Toast 문구 위치 확인 → `MainActivity.java`의 `onCreate`에서 `checkIntegrity()` 실패 시 `Toast` + `finish()`.
   - 문구가 미묘하게 달랐던 게 핵심 단서: JS는 "...실패했습니다. **앱을 종료합니다.**", 네이티브는 "...실패했습니다." → 실제로 본 건 **네이티브 쪽**.

### 결과 해석
무결성 검사는 **두 군데**에 있었고, 이전 세션엔 **JS 한 곳만** 고쳤다. 네이티브 검사가 WebView 로드보다 **먼저** 돌고 실패 시 `finish()`로 앱을 닫아버려서, 고쳐둔 JS 스킵 코드는 **실행될 기회조차 없었다.**

## 5. 원인 분석
- **확정 원인:** [MainActivity.java](platforms/android/app/src/main/java/com/trustnhope/vegas/MainActivity.java) `onCreate()`의 네이티브 무결성 검사가 debug 빌드를 막고 있었음. debug는 debug 키로 서명돼 서명 해시가 release 기대값과 달라 `verifySignature()`가 `false` → `Toast` + `finish()`.
  - 근거: logcat `MainActivity: Integrity check failed!`, Toast 문구가 네이티브 문자열과 일치, JS 스킵이 APK까지 들어갔는데도 실패.
- **이전 수정이 무력했던 이유:** JS 검사는 `deviceready`(WebView 로드 이후)에 돈다. 네이티브 검사는 그 **이전**인 `onCreate`에서 돌며 통과 못 하면 `finish()`. 따라서 WebView/JS까지 도달하지 못했음.

### 앱 시작 흐름 (어디서 갈렸나)
```
앱 실행
  │
  ▼ [네이티브] MainActivity.onCreate()          ← ① 가장 먼저
  │   checkIntegrity() = verifySignature()
  │   debug 서명 ≠ release 해시 → false
  │   Toast "앱 무결성 검증에 실패했습니다." + finish()  ← 여기서 종료(실제로 보던 것)
  │
  ▼ (①을 통과해야만) loadUrl("index.html")
  │
  ▼ [JS] deviceready → IntegrityManager.verifyOnLaunch()  ← ② 이전 세션에 고친 곳
      if (info.isDebug) skip ✅   (그러나 ①에서 막혀 도달 못 함)
```

## 6. 조치 내역
[MainActivity.java](platforms/android/app/src/main/java/com/trustnhope/vegas/MainActivity.java) `checkIntegrity()`에 `IntegrityPlugin`과 동일한 패턴의 debug 스킵 추가:

```java
// 개발(debug) 빌드는 debug 키로 서명되어 release 서명 해시와 달라
// 무결성 검사를 통과할 수 없다. chrome://inspect 디버깅을 위해 debug 빌드에서는
// 네이티브 검사를 건너뛴다. release 빌드(BuildConfig.DEBUG=false)는 기존대로 엄격히 검증.
if (BuildConfig.DEBUG) {
    Log.w(TAG, "Debug build detected - skipping native integrity check");
    return true;
}
// 서명 검증
return IntegrityChecker.verifySignature(this);
```
- `BuildConfig`는 같은 패키지(`com.trustnhope.vegas`)라 import 불필요(`IntegrityPlugin.java`와 동일).
- 서명 해시 로그(`App Signature Hash: ...`)는 스킵 위에 두어 debug에서도 그대로 출력되게 유지.
- release(`BuildConfig.DEBUG=false`)는 영향 없음.

이후 `./build.sh debug` 재빌드 → `adb install -r` 재설치.

## 7. 최종 상태
- 해결됨. 검증 로그(실기기):
```text
06-02 18:00:12  D MainActivity: App Signature Hash: C13B3A8D...0FB00EDF
06-02 18:00:12  D MainActivity: App Version: 1.1.29
06-02 18:00:12  W MainActivity: Debug build detected - skipping native integrity check
```
- "Integrity check failed!" 로그·Toast 사라짐, 앱 프로세스 생존(종료 안 됨), `chrome://inspect` 디버깅 가능.
- 남은 확인: [MainActivity.java](platforms/android/app/src/main/java/com/trustnhope/vegas/MainActivity.java)는 `platforms/android/` 안에만 존재(소스 원본 없음). `cordova platform rm/add android`로 플랫폼 재생성 시 이 수정이 사라짐 → 재적용 필요.

## 8. 학습 포인트
- 핵심 개념: 하이브리드(Cordova) 앱의 무결성/보안 가드는 **네이티브(Activity onCreate)** 와 **WebView(JS deviceready)** 두 계층에 따로 있을 수 있고, **네이티브가 먼저** 돈다. 한 계층만 고치면 다른 계층이 그대로 막는다.
- 이번에 배운 점: "코드/번들/APK까지 다 맞는데 증상 그대로"면 **같은 일을 하는 다른 계층**을 의심하라. logcat의 **태그(`MainActivity` vs `chromium/CONSOLE`)** 와 **메시지 문구의 미세한 차이**가 어느 계층인지 알려주는 결정적 단서.
- 다음에 같은 증상이면 확인할 순서: 1) `adb logcat`에서 실패 로그의 **태그**부터 본다(네이티브냐 JS냐) → 2) 문구를 `grep`해 발생 위치 특정 → 3) 그 계층의 분기(`BuildConfig.DEBUG` / `info.isDebug`) 확인.
- 👉 별도 학습 노트: [[2026-06-02-cordova-native-vs-webview-integrity]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 보안 가드/조기 종료(`finish()`, `exitApp()`) 로직은 **네이티브·JS 양쪽 모두** 점검했는가
  - [ ] debug 분기를 넣을 땐 두 계층에 **같은 기준**(`BuildConfig.DEBUG` ↔ `info.isDebug`)으로 넣었는가
  - [ ] 수정이 `platforms/` 전용이면 플랫폼 재생성 시 증발함을 인지/문서화했는가
- 추가하면 좋을 가드/로깅: 무결성 실패 시 **어느 계층에서 막았는지** 로그에 명시(이미 태그로 구분되지만 메시지에 `[native]`/`[js]` 표기 권장).
- 자동화 후보: `MainActivity.java`의 무결성 커스터마이징을 Cordova hook(`after_prepare`)이나 패치로 관리해 플랫폼 재생성에도 살아남게.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: debug 빌드인데 "앱 무결성 검증에 실패했습니다." + 앱 즉시 종료.
- 확인 절차:
  1. `adb logcat -c` 후 앱 재실행, `adb logcat -d -s MainActivity:*` 로 `Integrity check failed!` 여부 확인(네이티브 단서).
  2. 보이면 [MainActivity.java](platforms/android/app/src/main/java/com/trustnhope/vegas/MainActivity.java) `checkIntegrity()`의 debug 스킵 존재 확인.
  3. 안 보이고 WebView까지 뜨면 JS([www/js/integrity.js])의 `info.isDebug` 분기 확인.
- 조치 절차: 해당 계층에 `BuildConfig.DEBUG`(네이티브) / `info.isDebug`(JS) 스킵 추가 → `./build.sh debug` → 재설치.
- 정상 확인 기준: logcat에 `Debug build detected - skipping native integrity check` 출력 + 앱 프로세스 생존(`adb shell ps -A | grep vegas`).

## 11. 질문 기록

### 내가 던진 질문 (원문)
- "내가 이전 세션에서 debug 모등이면 무결성 검사를 안하기로 했는데 이게 무결성 검사가 되어서 앱 무결성 검즈에 실패했습니다 가 뜨고 있거든 어떻게 해야해?"
- "끄러면 이번에 js 고친 거랑 이거 굋ㄴ거랑 뭐가 다르고 어떤 흐름에서 달랐던거야 ? 플로우 다이어그램 그려줄래?"
- "그리고 /dev-log 에러메세지와 같이 그리고 어떻게 발견했는지 또한 어떻게 확인했는지 전체적인 과정이 있었으면 좋겠어"

### Claude가 되물은 확인 질문 + 답
- (이번 세션엔 되물은 확인 질문 없음 — logcat/grep/unzip으로 직접 사실 확인 후 진행)

## 12. 한 줄 결론
> 무결성 검사가 네이티브(`MainActivity.onCreate`)와 JS(`deviceready`) **두 계층**에 있었고, 네이티브가 먼저 `finish()`로 앱을 닫아 이전에 고친 JS 스킵이 무력했다 — `MainActivity.checkIntegrity()`에도 `BuildConfig.DEBUG` 스킵을 넣어 해결.
