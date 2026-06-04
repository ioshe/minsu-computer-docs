# Android Context로 얻는 정보 + 빌드 타입별로 바뀌는 값

> 한 줄 요약: `Context`(보통 `getApplicationContext()`)는 앱이 시스템/자기 자신에 접근하는 만능 손잡이다. 패키지·서명·버전·리소스·저장소·시스템 서비스 등을 꺼낼 수 있고, 그중 **서명 인증서 / versionName·Code / FLAG_DEBUGGABLE**은 빌드에 따라 바뀐다.

## 핵심
`Context`에서 대표적으로 꺼낼 수 있는 것:
- `getPackageManager()` — 패키지 정보, **서명 인증서**, 설치 앱 목록, 권한
- `getPackageName()` — applicationId (`com.trustnhope.vegas`)
- `getApplicationInfo()` — 앱 플래그(**FLAG_DEBUGGABLE** 등), targetSdk, 데이터 경로
- `getResources()` / `getAssets()` — 문자열·이미지·레이아웃, `www/` 에셋
- `getSharedPreferences()` — 로컬 key-value 저장소
- `getFilesDir()` / `getCacheDir()` — 앱 전용 파일/캐시 경로
- `getSystemService()` — 네트워크·위치·알림 등 시스템 서비스
- `getContentResolver()` — 연락처·미디어 등 ContentProvider

이 프로젝트의 무결성 플러그인이 실제로 쓰는 것: **서명 인증서, versionName, versionCode, packageName** (IntegrityChecker.java).

### 빌드 타입(debug/release)에 따라 바뀌는가
| 값 | 빌드 영향 | 어떻게 바뀌나 |
|----|-----------|---------------|
| 서명 인증서(해시) | 🔴 빌드 타입에 따라 바뀜 | debug=debug.keystore / release=내 .jks → 해시 다름 ([[2026-06-02-android-app-signing-cert-hash]]) |
| versionName / versionCode | 🟡 빌드마다 | config.xml → build.sh `sync_version` → build.gradle 복사 |
| **FLAG_DEBUGGABLE** | 🔴 빌드 타입에 따라 | debug=true / release=false (chrome://inspect 기준, [[2026-06-02-chrome-inspect-webview-debugging]]) |
| packageName(applicationId) | 🟢 거의 고정 | build.gradle `applicationId 'com.trustnhope.vegas'` (flavor/suffix 없으면 불변) |
| 리소스/에셋 | 🟡 내용물 | release는 build.sh가 JS 난독화 → www 에셋 내용 다름 |

## 언제 / 왜 쓰나
- 네이티브 플러그인에서 기기/앱 정보(서명·버전·권한·저장소)에 접근할 때 Context가 출발점.
- "debug/release에서 동작이 갈린다"의 원인 추적: Context에서 나오는 값 중 **빌드 종속(서명/플래그)**을 먼저 의심.

## 예시 (코드·명령)
```java
// Cordova 플러그인에서 Context 얻기 (cordova 필드는 프레임워크가 주입 — 아래 관련 노트)
Context context = cordova.getActivity().getApplicationContext();  // IntegrityPlugin.java L23
PackageManager pm = context.getPackageManager();
String pkg = context.getPackageName();                            // "com.trustnhope.vegas"
boolean debuggable = (context.getApplicationInfo().flags
                      & ApplicationInfo.FLAG_DEBUGGABLE) != 0;     // debug=true
```

## 흔한 함정 / 헷갈리는 점
- **`cordova` 필드와 `context`는 다른 것.** `cordova`(CordovaInterface)는 프레임워크가 `privateInitialize`에서 주입하고, `context`는 우리가 `execute()` 안에서 그 `cordova`로부터 꺼내 헬퍼에 넘긴다. (→ [[2026-06-02-cordova-js-java-bridge]])
- ApplicationContext vs Activity Context: 수명/용도 다름(UI/테마는 Activity context). 서명·패키지 조회는 ApplicationContext로 충분.
- versionName/packageName은 빌드 **타입**이 아니라 **설정(config.xml/build.gradle)**을 바꿔야 변함 → 디버깅 이슈와는 보통 무관.

## 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "cordova.getActivity().getApplicationContext()로 꺼낸 정보들은 어떤 정보들이 있는거야? 그리고 빌드할 때 어떤 영향을 받아서 바뀔 수도 있어?"

### Claude가 되물은 확인 질문 + 답
- (별도 확인 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-integrity-blocks-chrome-inspect]]
- 연결 노트: [[2026-06-02-cordova-js-java-bridge]] (cordova 필드/context 주입), [[2026-06-02-android-app-signing-cert-hash]], [[2026-06-02-chrome-inspect-webview-debugging]]

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02
