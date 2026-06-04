# AGP 8.x — BuildConfig는 opt-in (buildFeatures.buildConfig)

> 한 줄 요약: Android Gradle Plugin 8.x부터 `BuildConfig` 클래스는 기본 생성되지 않으며, `buildFeatures { buildConfig true }` 로 명시적으로 켜야 한다.

## 핵심
- AGP 7.x까지는 모든 모듈에 `BuildConfig` 클래스(`BuildConfig.DEBUG`, `BuildConfig.APPLICATION_ID` 등)가 자동 생성됐다.
- AGP 8.0부터 빌드 속도·불필요 생성 절감을 위해 여러 build feature가 **opt-in** 으로 바뀌었고, `BuildConfig` 도 그중 하나다.
- 따라서 코드에서 `BuildConfig.*` 를 참조하는데 모듈 build.gradle에 활성화가 없으면,
  컴파일 시 `error: cannot find symbol — variable BuildConfig` 가 난다.
- 이건 **import 문제가 아니다.** `BuildConfig` 는 앱 패키지(namespace)에 생성되므로 같은 패키지 코드는 import도 필요 없다.
  클래스 자체가 생성 안 된 게 문제.

## 언제 / 왜 쓰나
- 빌드 타입 분기(`if (BuildConfig.DEBUG)`), `BuildConfig.APPLICATION_ID`, `buildConfigField` 커스텀 상수를 쓸 때.
- AGP 8.x로 올라온(또는 그 위에서 새로 만든) 프로젝트에서 위를 처음 도입할 때 반드시 기능을 켜야 한다.

## 예시 (코드·명령)
```gradle
// app/build.gradle  (android { } 블록 안)
android {
    buildFeatures {
        buildConfig true
    }
}
```
프로젝트 전역으로 켜려면:
```properties
# gradle.properties
android.defaults.buildfeatures.buildconfig=true
```
확인:
```bash
grep -n "buildConfig\|buildFeatures" app/build.gradle
./gradlew :app:compileDebugJavaWithJavac   # BUILD SUCCESSFUL 이면 OK
```

## 흔한 함정 / 헷갈리는 점
- `cannot find symbol: BuildConfig` 를 보고 `import` 를 추가하려 하지만 소용없다 → build feature를 켜야 함.
- 같은 코드가 형제 프로젝트에선 빌드되는데 새 프로젝트에선 실패 → 형제 쪽 build.gradle엔 이미 플래그가 있는 경우.
- Cordova처럼 `platforms/android` 를 vendoring 해 다른 앱에 포팅할 때, Java 코드만 옮기고 build.gradle의 `buildFeatures`
  를 안 옮기면 이 에러가 난다.
- `viewBinding`, `compose`, `resValues` 등 다른 build feature도 동일하게 opt-in 이다.

## 질문 기록

### 내가 던진 질문 (원문)
- (이 학습 노트로 직접 이어진 사용자 질문 없음 — 빌드 에러 로그에서 파생)

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 기록할 질문 없음)

## 관련
- 발단이 된 이슈: [[2026-06-04-buildconfig-not-generated]]
- 참고 링크: Android Gradle Plugin 8.0 release notes — buildFeatures defaults

---
- 생성일: 2026-06-04
- 마지막 갱신: 2026-06-04
