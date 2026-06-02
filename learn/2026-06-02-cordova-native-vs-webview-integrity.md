# Cordova 하이브리드 앱의 이중 보안 가드 — 네이티브(Activity) vs WebView(JS)

> 한 줄 요약: Cordova 앱의 무결성/보안 검사는 네이티브 `Activity.onCreate`와 WebView의 JS(`deviceready`) **두 계층**에 따로 살 수 있고, 네이티브가 먼저 돈다 — 한 계층만 고치면 다른 계층이 그대로 막는다.

## 핵심
- Cordova 앱의 시작 순서는 **네이티브 → WebView → JS**다. `MainActivity.onCreate()`(네이티브)가 가장 먼저 실행되고, 그 안에서 `loadUrl("index.html")`로 WebView를 띄운 **뒤에야** `deviceready` 이벤트가 발생하고 JS가 돈다.
- 그래서 `onCreate`에서 `finish()`(또는 조건 분기)로 막아버리면 **WebView/JS 코드는 아예 실행되지 않는다.** JS 쪽에 아무리 좋은 분기를 넣어도 도달 자체를 못 한다.
- 보안 가드(무결성/루팅/서명 검증 등)는 흔히 **양쪽에 중복**으로 들어간다(네이티브 1차, JS 2차). 그래서 동작을 바꾸려면 **두 계층을 같은 기준으로** 함께 손봐야 한다.
- 어느 계층이 막았는지는 **logcat 태그**로 구분된다: 네이티브는 `MainActivity`/커스텀 태그, JS `console.log`는 `chromium` 또는 `Web Console`/`CONSOLE` 태그로 찍힌다.

## 언제 / 왜 쓰나
- "코드도 맞고 번들도 맞고 APK 안까지 들어갔는데 증상이 그대로"일 때 → **같은 일을 하는 다른 계층**을 의심하는 사고 모델.
- debug 빌드에서만 보안 가드를 풀어 `chrome://inspect` 디버깅을 열고 싶을 때: 네이티브엔 `BuildConfig.DEBUG`, JS엔 플러그인이 내려주는 `isDebug`(= `BuildConfig.DEBUG`)로 **양쪽** 분기.

## 예시 (코드·명령)
```java
// [네이티브] MainActivity.checkIntegrity() — 1차, onCreate에서 가장 먼저
if (BuildConfig.DEBUG) {          // 같은 패키지면 import 불필요
    Log.w(TAG, "Debug build detected - skipping native integrity check");
    return true;
}
return IntegrityChecker.verifySignature(this);
```
```javascript
// [JS] integrity.js — 2차, deviceready 이후 (네이티브 통과해야 도달)
if (info.isDebug) {               // 네이티브 플러그인이 BuildConfig.DEBUG를 내려줌
    onSuccess({ skipped: true, reason: 'debug_build' });
    return;
}
```
```bash
# 어느 계층이 막았는지 태그로 판별
adb logcat -c && adb shell am force-stop <pkg> && adb shell monkey -p <pkg> -c android.intent.category.LAUNCHER 1
adb logcat -d -s MainActivity:*        # 네이티브 실패 단서: "Integrity check failed!"
adb logcat -d | grep -iE "chromium|CONSOLE"   # JS 콘솔 단서
adb shell ps -A | grep <pkg>           # 프로세스 생존 = 종료 안 됨(정상)
```

## 흔한 함정 / 헷갈리는 점
- **"JS 고쳤는데 왜 그대로지?"** → 네이티브가 `onCreate`에서 먼저 `finish()`하면 JS는 실행조차 안 된다. JS는 항상 WebView 로드 *이후*다.
- **문구가 거의 같아서 헷갈림** — 네이티브 Toast "…실패했습니다."와 JS alert "…실패했습니다. **앱을 종료합니다.**"처럼 끝부분만 다른 경우, 문구 차이가 **어느 계층인지** 알려주는 단서. `grep`으로 문자열 위치를 찾아 계층 특정.
- **`BuildConfig` import** — Activity/플러그인이 앱과 같은 패키지(namespace)면 import 없이 `BuildConfig.DEBUG` 사용 가능. 다른 패키지의 `BuildConfig`를 잘못 import하면 라이브러리용이라 항상 false가 되는 고전 함정.
- **`platforms/` 직접 수정의 휘발성** — `MainActivity.java`처럼 소스 원본 없이 `platforms/android/`에만 있는 파일을 고치면 `cordova platform rm/add`로 증발. 영구화하려면 Cordova hook(`after_prepare`)이나 패치로 관리.

## 질문 기록

### 내가 던진 질문 (원문)
- "끄러면 이번에 js 고친 거랑 이거 굋ㄴ거랑 뭐가 다르고 어떤 흐름에서 달랐던거야 ?"

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 기록할 되물은 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-native-integrity-check]]
- 참고: Cordova Android `CordovaActivity` 생명주기, `BuildConfig.DEBUG`

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02
