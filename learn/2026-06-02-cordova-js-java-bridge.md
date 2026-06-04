# Cordova JS ↔ Java 브리지 동작 원리 (cordova.exec)

> 한 줄 요약: WebView의 JS는 `cordova.exec`로 "서비스 이름 + 액션 + 콜백번호표"를 문자열로 네이티브에 던지고, 네이티브는 config.xml의 feature 매핑으로 Java 클래스를 찾아 실행한 뒤, 그 번호표(callbackId)를 들고 결과를 JS 콜백으로 돌려보낸다. `return`이 아니라 비동기 콜백으로 값이 온다.

## 핵심

Cordova 앱은 **네이티브 안드로이드(Java) 앱이 주인이고, 그 안에 WebView를 띄워 HTML/JS를 보여주는** 구조다. JS는 기기 기능(서명, 파일, 권한 등)에 직접 접근 못 하므로, **JS → Java로 건너가는 다리**가 필요하다. 그 다리가 `cordova.exec` 하나다.

전체 한 줄 흐름:
```
JS: AppIntegrity.getIntegrityInfo(cb)
  → cordova.exec(cb, err, "IntegrityPlugin", "getIntegrityInfo", [])
  → [JavascriptInterface] SystemExposedJsApi.exec(...)
  → CordovaBridge.jsExec(...)
  → PluginManager.exec("IntegrityPlugin", ...)   // config.xml로 클래스 찾기
  → IntegrityPlugin.execute("getIntegrityInfo", args, callbackContext)
  → callbackContext.success(result)              // 번호표 들고 JS로 복귀
  → JS cb(result) 실행
```

핵심 개념 3가지:
1. **이름 매칭은 자동이 아니라 config.xml의 `<feature>` 테이블 덕분**이다. ("IntegrityPlugin" 문자열 → `com.trustnhope.vegas.IntegrityPlugin` 클래스)
2. **`callbackId`(번호표)** 가 JS 콜백 함수의 주소 역할을 한다. 동기 `return`이 아니라 이 번호표로 비동기 배달.
3. **`cordova` 필드**는 우리가 만든 게 아니라 프레임워크가 플러그인 생성 직후 `privateInitialize`로 꽂아준다.

---

## Q1. cordova.exec — 현재 코드에서 실제 구현 (JS쪽)

`cordova.exec`의 실체는 빌드된 `cordova.js` 안의 `androidExec`다.
`platforms/android/app/src/main/assets/www/cordova.js:933`

```javascript
function androidExec(success, fail, service, action, args) {
    ...
    // ① 콜백 번호표 생성: 서비스명 + 증가하는 카운터
    var callbackId = service + cordova.callbackId++,
        argsJson = JSON.stringify(args);

    // ② 번호표 → {성공콜백, 실패콜백} 을 JS측 테이블에 등록해 둠
    if (success || fail) {
        cordova.callbacks[callbackId] = {success:success, fail:fail};
    }

    // ③ 네이티브로 문자열들을 던짐 (여기서 JS세계 → Java세계로 넘어감)
    var msgs = nativeApiProvider.get().exec(bridgeSecret, service, action, callbackId, argsJson);
    ...
}
```
(`cordova.exec = require('cordova/exec')` — cordova.js:1336)

포인트:
- 넘기는 건 전부 **문자열/숫자**: `service`("IntegrityPlugin"), `action`("getIntegrityInfo"), `callbackId`, `argsJson`. 함수(success/fail)는 안 넘어간다 — 대신 `cordova.callbacks[callbackId]`에 보관.
- `nativeApiProvider.get().exec(...)`는 OS가 WebView에 심어둔 네이티브 객체의 메서드 호출 = **다리를 건너는 순간**.

우리 플러그인 JS는 이걸 얇게 감싼 것뿐이다.
`plugins/cordova-plugin-integrity/www/integrity.js`
```javascript
var exec = require('cordova/exec');
var AppIntegrity = {
    getIntegrityInfo: function(success, error) {
        exec(success, error, 'IntegrityPlugin', 'getIntegrityInfo', []);
        //                    └ service           └ action
    }
};
```

## Q2. JS의 AppIntegrity가 어떻게 Java를 호출하나 — 매칭은 자동인가?

**자동으로 폴더(`plugins/cordova-plugin-integrity`)를 읽는 게 아니다.** 두 개의 "전화번호부"가 빌드 시점에 만들어져 매칭을 가능하게 한다.

### (a) JS 전역 이름 `AppIntegrity`는 어디서? → cordova_plugins.js의 `clobbers`
`platforms/android/app/src/main/assets/www/cordova_plugins.js`
```javascript
{
    "id": "cordova-plugin-integrity.integrity",
    "file": "plugins/cordova-plugin-integrity/www/integrity.js",
    "pluginId": "cordova-plugin-integrity",
    "clobbers": ["AppIntegrity"]    // ← 이 모듈을 window.AppIntegrity 에 꽂아라
}
```
→ 그래서 JS 어디서든 `AppIntegrity.getIntegrityInfo(...)`가 된다.

### (b) "IntegrityPlugin" 문자열 → Java 클래스는 어디서? → config.xml의 `<feature>`
`platforms/android/app/src/main/res/xml/config.xml:46`
```xml
<feature name="IntegrityPlugin">
    <param name="android-package" value="com.trustnhope.vegas.IntegrityPlugin" />
</feature>
```
→ 네이티브의 `PluginManager`가 앱 시작 시 이 feature 목록을 읽어 `service이름 → 클래스` 테이블(entryMap)을 만든다. `exec`의 service 문자열로 이 테이블을 조회해 클래스를 찾는다.

`PluginManager.getPlugin(service)` (platforms/android/CordovaLib/.../PluginManager.java:159)
```java
PluginEntry pe = entryMap.get(service);          // config.xml에서 만든 테이블 조회
ret = instantiatePlugin(pe.pluginClass);         // 클래스 인스턴스 생성(최초 1회)
ret.privateInitialize(service, ctx, app, ...);   // ← 여기서 cordova 필드 주입 (Q3)
```

> 이 프로젝트는 표준 cordova가 아니라 **build.sh가 빌드 때 위 두 곳(config.xml feature + cordova_plugins.js clobbers)을 직접 주입**한다.
> 즉 "매칭"은 마법이 아니라 **빌드 스크립트가 미리 등록해 둔 두 테이블** 덕분.

### 네이티브 다리의 실제 진입점 (JavascriptInterface)
`SystemExposedJsApi.java:39`
```java
@JavascriptInterface                                 // ← JS가 부를 수 있게 노출
public String exec(int bridgeSecret, String service, String action,
                   String callbackId, String arguments) {
    return bridge.jsExec(bridgeSecret, service, action, callbackId, arguments);
}
```
→ `CordovaBridge.jsExec`(CordovaBridge.java:44) → `pluginManager.exec(service, action, callbackId, arguments)`(line 59).

`PluginManager.exec` (PluginManager.java:121)
```java
public void exec(String service, String action, String callbackId, String rawArgs) {
    CordovaPlugin plugin = getPlugin(service);                  // ① 클래스 찾기
    CallbackContext callbackContext = new CallbackContext(callbackId, app); // ② 번호표 포장
    boolean wasValidAction = plugin.execute(action, rawArgs, callbackContext); // ③ 실행
    if (!wasValidAction) { ... INVALID_ACTION ... }             // execute가 false 반환 시
}
```
→ `CallbackContext`가 `callbackId`를 들고 있다가, 나중에 `success()/error()`로 그 번호표에 맞는 **JS 콜백**을 깨운다.

## Q3. `cordova` 필드는 누가 어떻게 채우나 — Cordova 처음 보는 사람용

문제의 코드 (우리 플러그인):
```java
public class IntegrityPlugin extends CordovaPlugin {
    public boolean execute(String action, JSONArray args, CallbackContext callbackContext) {
        Context context = cordova.getActivity().getApplicationContext();  // ← 이 cordova가 뭐냐?
        ...
    }
}
```

`cordova`는 **우리가 선언한 적 없는 변수**다. 부모 클래스 `CordovaPlugin`이 갖고 있는 **public 필드**다.

`CordovaPlugin.java:42`
```java
public CordovaWebView webView;
public CordovaInterface cordova;     // ← 상속받은 필드. 자식(IntegrityPlugin)에서 그냥 cordova 로 접근
```

그럼 이 필드에 **값은 누가 넣나?** → 우리가 아니라 **프레임워크**가 넣는다.
플러그인이 처음 필요해지는 순간(`getPlugin`)에 `privateInitialize`가 호출되며 주입된다.

`CordovaPlugin.java:51`
```java
public final void privateInitialize(String serviceName, CordovaInterface cordova,
                                    CordovaWebView webView, CordovaPreferences preferences) {
    assert this.cordova == null;
    this.cordova = cordova;       // ← 여기서 필드가 채워짐 (프레임워크가 자기 자신을 넘겨줌)
    this.webView = webView;
    ...
}
```

### "처음 보는 사람" 버전으로 다시
1. 안드로이드 앱은 **Activity**(화면을 가진 앱의 실체)와 **Context**(앱 환경에 접근하는 손잡이)가 있어야 시스템 기능을 쓴다. 우리 플러그인은 서명 해시를 읽으려면 `Context`가 필요하다.
2. 하지만 플러그인은 그 Context를 **직접 만들 수 없다.** 앱 전체를 관리하는 누군가에게 받아야 한다.
3. 그 "누군가"가 `CordovaInterface`(= `cordova` 필드)다. 앱의 Activity로 가는 통로.
4. Cordova가 플러그인 객체를 만든 직후 `privateInitialize`로 **자기 자신(cordova)을 플러그인에 끼워준다.** 그래서 우리는 선언/대입 안 했는데도 `cordova`를 바로 쓸 수 있다.
5. 우리는 그 `cordova`에서 `getActivity().getApplicationContext()`로 Context를 꺼내, 우리가 만든 헬퍼 `getIntegrityInfo(context, callbackContext)`에 **인자로 직접 전달**한다.

> 비유: `cordova`는 입사 첫날 회사가 책상에 놔준 **사원증**이다. 내가 만들지 않았지만(프레임워크가 줌), 그 사원증으로 사내 시설(Activity/Context)에 들어간다.

---

## 언제 / 왜 쓰나

- Cordova/하이브리드 앱에서 JS가 못 하는 일(네이티브 API: 서명·파일·권한·디바이스)을 해야 할 때.
- 커스텀 네이티브 플러그인을 만들거나 디버깅할 때 "JS 호출이 왜 Java에 안 닿지?"를 추적할 때 → config.xml feature와 cordova_plugins.js clobbers부터 확인.

## 흔한 함정 / 헷갈리는 점

- **`execute`의 `return true/false`는 데이터 반환이 아니다.** "이 action 처리할 줄 안다(true)/모른다(false)"는 접수 영수증일 뿐. false면 Cordova가 `INVALID_ACTION`을 JS error로 보냄(PluginManager.java:138). 실제 데이터는 **`callbackContext.success(result)`** 로만 간다.
- **같은 이름의 함수가 계층별로 둘 있을 수 있다.** 예: `IntegrityPlugin.verifySignature(context, callbackContext)`(JS용 배달부) vs `IntegrityChecker.verifySignature(context)`(순수 로직). 이름만 같고 역할이 다르다.
- **매칭은 자동 폴더 스캔이 아니다.** config.xml `<feature>`와 cordova_plugins.js에 등록돼 있어야 한다. 이 프로젝트는 build.sh가 빌드 때 주입하므로, **원본 plugins/ 만 고치고 빌드 안 하면 반영 안 됨.**
- `cordova` 필드는 **플러그인이 처음 호출될 때** 채워진다(lazy). 생성자에서 `cordova`를 쓰면 아직 null일 수 있다.

## 보충 — 두 번 헷갈린 지점 (다시 물어본 것들)

### (1) "verifySignature가 미사용"이라는 말의 정확한 범위
이름이 같은 `verifySignature`가 **2개**라 혼동하기 쉽다.

| 함수 | 정체 | 사용 |
|------|------|------|
| `IntegrityChecker.verifySignature(context)` (Checker.java:75) | 해시 비교 **로직(두뇌)** | ✅ 사용됨 — `getIntegrityInfo`가 Plugin.java:47에서 호출 |
| `IntegrityPlugin.verifySignature(context, callbackContext)` (Plugin.java:68) | JS용 **배달부(액션 핸들러)** | ❌ 미사용 — JS가 이 액션을 안 부름 |

- "안 쓰인다"고 한 건 **후자(Plugin의 verifySignature, line 68)**다.
- JS 흐름은 `AppIntegrity.getIntegrityInfo()` 액션만 부르고(`AppIntegrity.verifySignature()`는 안 부름), 그 getIntegrityInfo가 내부에서 **두뇌(Checker.verifySignature)를 직접** 호출한다.
- 그래서 구조는 **로직 1개(Checker) + 그걸 JS에 노출하는 배달부 2개(getIntegrityInfo ✅, verifySignature ❌)**. 두뇌는 당연히 쓰이고, 배달부 하나가 놀고 있는 것.

### (2) context는 언제/누가 넣어주나 + 해시 탈락까지 전체 플로우
**`cordova` 필드와 `context`는 다른 것**이고 시점도 다르다.
- `cordova` 필드 → **프레임워크가** 플러그인 최초 호출 시 `privateInitialize`에서 주입 (Plugin.java:54)
- `context` → 프레임워크가 직접 안 줌. **우리가** `execute()` 안에서 `cordova` 필드로부터 꺼내 헬퍼에 인자로 전달 (IntegrityPlugin.java:23)

```
[앱 실행 / deviceready]  (JS)
   AppIntegrity.getIntegrityInfo(cb)
      → cordova.exec(cb, err, "IntegrityPlugin", "getIntegrityInfo", [])
   ───────── JS세계 → Java세계 (다리) ─────────
   PluginManager.exec("IntegrityPlugin", ...)
      ├ getPlugin("IntegrityPlugin")
      │    └(최초 1회) 인스턴스 생성 + privateInitialize(...)
      │         └ ★ this.cordova = cordova        ← [cordova 필드 주입 시점]
      ▼
   plugin.execute("getIntegrityInfo", args, callbackContext)
      │  // execute() 안:
      │  Context context = cordova.getActivity().getApplicationContext();
      │         └ ★ [context를 cordova 필드로부터 꺼냄]
      ▼
   getIntegrityInfo(context, callbackContext)     ← ★ [우리가 context를 인자로 전달]
      ▼
   IntegrityChecker.verifySignature(context)      ← 두뇌 (위 (1)의 ✅)
      ├ getAppSignatureHash(context)
      │    └ context.getPackageManager()로 현재 APK 서명 추출 → SHA-256
      │       = "현재 해시" (debug: c13b3a8d... / 실시간 계산)
      ▼
   return EXPECTED_SIGNATURE_HASH.equals(currentHash)
      └ "하드코딩 해시" = IntegrityChecker.java:17 = 8B717936...
      └ debug: 8B7179... ≠ c13b3a8d... → false    ← [탈락 지점]
      ▼
   callbackContext.success({isValid:false, ...})  ── 번호표 들고 JS로 복귀
      ▼
   JS: if(!info.isValid) → onFailure → exitApp()  ← [debug 앱 종료]
```

핵심 3줄:
1. `context`는 프레임워크가 따로 주입하는 게 아니라, `execute()` 안에서 **우리가 `cordova` 필드로부터 꺼내** 헬퍼에 인자로 넘긴다. (`cordova` 필드만 프레임워크가 주입)
2. 그 `context`로 OS에서 **현재 서명 해시(debug=c13b3a8d…)**를 실시간 계산한다.
3. **하드코딩 해시(8B7179…, IntegrityChecker.java:17)**와 다르면 `false` → debug는 여기서 탈락 → JS가 그 false를 보고 앱 종료.

## 질문 기록
> 원칙 8. (스킬 갱신으로 사후 추가)

### 내가 던진 질문 (원문)
- "해당 자바 코드는 그 native?라고 하는건가 os 에서 제공하는 함수인건가? 아니며 우리가 만든건가,,, 흐음 그리고 이거 다이어그램으로 요청 흐름이 어떻게 되는거지 어디에 요청을 보내는거고 어디서 요청을 하는거고 무결성 검사는 어디에서 하는거고, 어떻게 걸리는거고,,,흐음"
- "1. 아니 그러면 이 앱이 뜰 떄는 java engine 이 같이 뜨는거야>? 그럼 엄청 무거운거아닌가,,? 2. 그리고 cordova.exec (JS→Java 다리)라고 하는데 이 부분을 보다 자세히 설명해줬으면 좋겠어 3. AppIntegrity 는 js 에서 java 를 호출하는데 이게 잘 모르겠는데,, 어케 가능한거임..? 2번의 연장선이잖아 잘 설명해줘봐 4. java 에서는 private boolean getIntegrityInfo(Context context, CallbackContext callbackContext) { 와 같이 받고 있는데 혹시, context 는 누가 보내주고 그 인자들은 어떻게 아는건지 모르겠네? 5. callbackContext.success(result); 이 함수의 정체는 뭐야 ... 어떻게 함수 밖에서 info.isValid 를 할 수 있는거야 ? 주소를 전달하는 과정인가? ... verifyWithServer ... if (response.verified === false) 여기서 걸린 거 같다 ... test 일 경우는 넘기도록 하는게 좋아보이는데 어떻게 생각하니,,"
- "1. cordova.exec 에서는 현재 코드에서는 어떻게 구현되어 있는지 예시를 실제 코드를 보여주면 하 ㄹ수 있나. 2. JS의 AppIntegrity가 어떻게 Java를 호출하나? ... 파일 규칙을 자동으로 읽는거야? 어떻게 이게 매칭되는지 보다 자세히 알고 싶은데 3. cordova는 부모 CordovaPlugin이 들고 있는 필드예요 ... 이 부분이 너무 이해가 안되는데 보다 자세히,,, 설명해줘 cordova 처음 보는 사람도 이해할 수 잇을 정도,"
- "1. 왜 verifySignature를 또 따로 만들었나? 에서 getIntegrityInfo를 쓰니까 지금은 사실상 미사용라고 해쓴ㄴ데 getIntegrityInfo 안에서 boolean isValid = IntegrityChecker.verifySignature(context); 이 줄 이 있으니까 ... 근데 왜 안쓴다고 하는거녀ㅑ,ㅡㅡ,,,,,, 2. ... context 는 언제 넣어준다고 햇었지? 그 플로우 다이어그램이랑 같이 넣어줄래 ... 그 전 히스토리 맥락도 잇었으면 좋겠어"

### Claude가 되물은 확인 질문 + 답
- (개념 설명 위주라 별도의 예/아니오 확인 질문은 없었음. 매 답변 끝에 "학습 노트로 남길까요 / 우회 수정을 적용할까요?"를 제안했고, 사용자는 이후 "일단 우회 수정을 하자"로 진행)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-native-integrity-check]] (debug 빌드 무결성 검사가 chrome://inspect 디버깅을 막던 건)
- 실제 코드 위치:
  - JS exec: `platforms/android/app/src/main/assets/www/cordova.js:933`
  - 다리 진입점: `CordovaLib/.../engine/SystemExposedJsApi.java:39`
  - 라우팅: `CordovaLib/.../PluginManager.java:121, 159`
  - 필드 주입: `CordovaLib/.../CordovaPlugin.java:42, 51`
  - feature 매핑: `platforms/android/app/src/main/res/xml/config.xml:46`

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02 (보충: verifySignature 2개 구분 / context 주입 시점 + 해시 탈락 플로우 / 질문 기록 항목 추가)
