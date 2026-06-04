# chrome://inspect로 WebView 원격 디버깅이 뜨는 기준 (FLAG_DEBUGGABLE)

> 한 줄 요약: 안드로이드 WebView가 `chrome://inspect`에 뜨려면 네이티브에서 `WebView.setWebContentsDebuggingEnabled(true)`가 호출돼야 하고, Cordova는 이걸 **앱이 debuggable(=debug 빌드)일 때만** 자동 호출한다. 그래서 debug는 뜨고 release는 안 뜬다 — 단, 앱이 살아있어야 한다.

## 핵심
- chrome://inspect 노출 조건 = WebView에 `setWebContentsDebuggingEnabled(true)`가 켜져 있을 것.
- Cordova(cordova-android)는 `SystemWebViewEngine`에서 **`ApplicationInfo.FLAG_DEBUGGABLE`가 켜진 경우에만** 이 호출을 한다.
- `FLAG_DEBUGGABLE`은 **debug 빌드에만** 자동으로 붙고 release엔 안 붙는다.
  - debug → debuggable → chrome://inspect 뜸 ✅
  - release → not debuggable → 안 뜸 ❌ (앱 코드에서 명시적으로 켜지 않는 한)
- 추가 조건: PC Chrome `chrome://inspect/#devices`, 기기 USB 디버깅 ON, 데이터 케이블 연결, **그리고 앱이 실행 중이어야 함**(앱이 죽으면 타깃도 사라짐).

## 언제 / 왜 쓰나
- 하이브리드(Cordova/Capacitor) 앱의 WebView 안 JS/DOM을 PC Chrome 개발자도구로 디버깅할 때.
- "분명 debug인데 chrome://inspect에 안 뜬다" 진단: 앱이 실행 중인지, 시작 직후 무결성/검사 로직으로 종료되는지부터 확인.

## 예시 (코드·명령)
```java
// platforms/android/CordovaLib/.../engine/SystemWebViewEngine.java
if ((appInfo.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {   // L176
    enableRemoteDebugging();                                    // L177
}
...
private void enableRemoteDebugging() {                          // L224
    WebView.setWebContentsDebuggingEnabled(true);               // L226
}
```
```text
PC: chrome://inspect/#devices  →  기기/앱 WebView 타깃 → inspect
```

## 흔한 함정 / 헷갈리는 점
- **debug 빌드인데 안 뜬다 → 보통 "앱이 떠 있지 않다".** 이번 프로젝트는 무결성 검사가 debug에서 실패해 `exitApp()`으로 앱이 즉시 종료 → 타깃이 잠깐 떴다 사라졌다. (→ [[2026-06-02-debug-integrity-blocks-chrome-inspect]])
- **release에서 디버깅하려고 무결성을 풀어도 소용없다.** release는 애초에 debuggable이 아니라 chrome://inspect에 안 뜬다. 디버깅하려면 debug 빌드가 필요.
- 따라서 "디버깅하려면 debug 필요한데 debug는 무결성에서 죽는다"는 딜레마가 생긴다 → debug에서 무결성 스킵으로 해소.

## 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "내가 하고 싶은 것은 chrome:inspect 에서 debuging 하고 싶은건데 이거 뜨게 하는 기준이 뭔지 알려줘 그리고 현재 상태 스크립트 점검하고"

### Claude가 되물은 확인 질문 + 답
- (별도 확인 질문 없음 — CordovaLib 소스에서 조건을 확인해 답하고 build.sh도 함께 점검)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-integrity-blocks-chrome-inspect]]
- 연결 노트: [[2026-06-02-android-context-info]] (FLAG_DEBUGGABLE은 ApplicationInfo.flags)
- 참고 링크: WebView.setWebContentsDebuggingEnabled (Android), chrome://inspect

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02
