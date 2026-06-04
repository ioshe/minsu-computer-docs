# 2026-06-04 · 디버그 빌드 실패: BuildConfig.DEBUG `cannot find symbol`

## 1. 개요
- 날짜: 2026-06-04
- 프로젝트 / 화면 / 모듈: hanchartAppLite-app / 앱 무결성 플러그인(cordova-plugin-integrity)
- 이슈 유형: 빌드 실패 (컴파일 에러)
- 계층: 빌드·배포 (Android Gradle Plugin 8.x 설정)
- 최종 상태: 해결
- 중요도: 높음 (디버그 빌드 자체가 안 됨)

## 2. 환경
- OS / 기기 / 웹뷰: Ubuntu(Linux) / Android 빌드
- 앱 버전 / 빌드: debug 빌드 (`./build.sh debug` → `./gradlew clean assembleDebug`)
- 브랜치 / 커밋: feature/isms (수정 전 HEAD `a395e73`)
- 관련 의존성·버전: Gradle 8.7, AGP 8.x, JDK 17, cordova-android 7.1.x, compileSdk 35, buildTools 35.0.0
- 패키지: `com.trustnhope.hanchartapplite`

## 3. 증상
### 관찰된 현상
- vegas의 "debug 빌드에서 무결성 검사 건너뛰기" 기능을 hanchart에 포팅하면서 `IntegrityPlugin.java`에
  `result.put("isDebug", BuildConfig.DEBUG);` 한 줄을 추가했더니, 디버그 빌드가 `:app:compileDebugJavaWithJavac`
  단계에서 컴파일 실패했다.

### 재현 절차
1. `IntegrityPlugin.java`에 `BuildConfig.DEBUG` 참조 추가
2. `./build.sh debug` 실행 (내부에서 `./gradlew clean assembleDebug`)
3. `:app:compileDebugJavaWithJavac` 에서 FAILED

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
> Task :app:compileDebugJavaWithJavac FAILED
/home/minsu/src/hanchartAppLite-app/platforms/android/app/src/main/java/com/trustnhope/hanchartapplite/IntegrityPlugin.java:53: error: cannot find symbol
            result.put("isDebug", BuildConfig.DEBUG);
                                  ^
  symbol:   variable BuildConfig
  location: class IntegrityPlugin
1 error

FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':app:compileDebugJavaWithJavac'.
> Compilation failed; see the compiler error output for details.
BUILD FAILED in 23s
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점:
  - `grep -rn "buildConfig\|buildFeatures"` 로 vegas와 hanchart의 build.gradle / gradle.properties 비교
  - vegas: `platforms/android/app/build.gradle:188-189` 에 `buildFeatures { buildConfig true }` **있음**
  - hanchart: 동일 설정 **없음**
  - 양쪽 모두 `.java`에 `import ...BuildConfig` 는 없음 (같은 패키지라 import 불필요 — import가 원인이 아님)
- 결과 해석:
  - vegas는 빌드가 됐고 hanchart만 실패 → 차이는 `BuildConfig` 클래스의 **생성 여부**. import 문제·패키지 문제가 아니라
    BuildConfig 클래스가 애초에 만들어지지 않은 것.

## 5. 원인 분석
- **확정 원인:** Android Gradle Plugin 8.x 부터 `BuildConfig` 클래스 자동 생성이 **기본 비활성화(opt-in)** 되었다.
  hanchart의 `app/build.gradle` 에 `buildFeatures { buildConfig true }` 가 없어 `com.trustnhope.hanchartapplite.BuildConfig`
  자체가 생성되지 않았고, 그래서 `BuildConfig.DEBUG` 가 `cannot find symbol` 로 실패했다.
  (vegas는 같은 줄을 쓰고도 빌드된 이유 = vegas build.gradle엔 이미 해당 플래그가 켜져 있었기 때문.)

## 6. 조치 내역
- 근본 수정(root fix): `platforms/android/app/build.gradle` 의 `android { }` 블록(`lintOptions` 다음, `compileSdk` 앞)에
  vegas와 동일하게 추가.
  ```gradle
  buildFeatures {
      buildConfig true
  }
  ```
- 조치 결과:
  - `./gradlew :app:compileDebugJavaWithJavac` → **BUILD SUCCESSFUL** (BuildConfig.DEBUG 정상 해석)
- 커밋: `f97ec14` "build.gradle: buildConfig 기능 활성화 (BuildConfig.DEBUG 생성)" → origin/feature/isms push 완료

## 7. 최종 상태
- 해결 여부: 해결 (컴파일 통과 확인)
- 남은 문제 / 추가 확인 필요: 전체 `assembleDebug` 로 APK 산출까지는 미실행 — 컴파일 단계만 검증함.

## 8. 학습 포인트
- 핵심 개념: AGP 8.x의 build feature opt-in — `BuildConfig`/`viewBinding` 등은 명시적으로 켜야 생성된다.
- 이번에 배운 점: `cannot find symbol: BuildConfig` 는 import 누락이 아니라 **build feature 미활성화**일 수 있다.
- 다음에 같은 증상이면 확인할 순서:
  1) `import` 문제인지 (같은 패키지면 import 불필요)
  2) `app/build.gradle` 에 `buildFeatures { buildConfig true }` 가 있는지
  3) `gradle.properties` 의 `android.defaults.buildfeatures.buildconfig` 값
  4) 정상 동작하는 형제 프로젝트(vegas)와 build.gradle diff
- 👉 별도 학습 노트: [[2026-06-04-agp8-buildconfig-feature]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 코드에서 `BuildConfig.*` 를 새로 참조하면, 해당 모듈 build.gradle에 `buildConfig true` 가 있는지 먼저 확인
  - [ ] vendored `platforms/android` 를 다른 앱에 포팅할 때 build.gradle의 `buildFeatures` 도 함께 옮겼는지 확인
- 추가하면 좋을 가드: 형제 프로젝트(vegas ↔ hanchart) 간 build.gradle 핵심 설정 diff를 포팅 시 1회 비교
- 자동화 후보: 없음 (설정 1회성)

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `:app:compileDebugJavaWithJavac` 에서 `error: cannot find symbol ... variable BuildConfig`
- 확인 절차: `grep -n "buildConfig\|buildFeatures" platforms/android/app/build.gradle`
- 조치 절차: `android { }` 블록에 `buildFeatures { buildConfig true }` 추가
- 정상 확인 기준: `./gradlew :app:compileDebugJavaWithJavac` → BUILD SUCCESSFUL

## 11. 질문 기록

### 내가 던진 질문 (원문)
- "이와 같게 /home/minsu/src/hanchartAppLite-app 도 수정해줄래? 구조가 같아"
- "그러면 빌드 다시 하면 사라지는거 아니야??,,??? ㅇㅇ 확인해줄래"
- "build.sh 도 vegas-app-lite 현재 우분투에서도 mac os에서도 되게 수정했었는데 그 부분도 수정해줘"
- (이 이슈는 사용자가 빌드 에러 로그 전문을 붙여넣어 시작됨 — 별도 질문 문장은 없음)

### Claude가 되물은 확인 질문 + 답
- Q: hanchart MainActivity는 vegas와 달리 네이티브 무결성 검사가 없는데 어떻게 할까?
  - A: "JS 검사만 유지 (권장)" — MainActivity는 순정 그대로 둠
- Q: vegas의 push된 minsu 커밋 3개 작성자를 ioshe로 재작성(force-push)할까?
  - A: "그대로 두기 (권장)" — 재작성/force-push 안 함

## 12. 한 줄 결론
> AGP 8.x에서 `BuildConfig.DEBUG`가 `cannot find symbol`이면 import가 아니라 `buildFeatures { buildConfig true }` 누락을 의심하라.
