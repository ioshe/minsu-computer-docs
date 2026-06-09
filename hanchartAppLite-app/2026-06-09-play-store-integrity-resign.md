# 2026-06-09 · Play Store 배포 시에만 무결성 검사 실패 — Google 재서명으로 서명 해시가 바뀜

## 1. 개요
- 날짜: 2026-06-09
- 프로젝트 / 화면 / 모듈: hanchartAppLite-app / 앱 시작 무결성 검사 (cordova-plugin-integrity)
- 이슈 유형: 빌드·배포 (서명/패키징)
- 계층: 빌드·배포 + 네이티브·플러그인
- 최종 상태: 해결
- 중요도: 높음 (스토어 배포 시 정상 사용자가 앱 진입 차단됨)

## 2. 환경
- OS / 기기 / 웹뷰: Android (실기기), Cordova WebView 하이브리드
- 앱 버전 / 빌드: release 빌드 (`./build.sh release`)
- 브랜치 / 커밋: feature/isms
- 관련 의존성: cordova-plugin-integrity (자체 플러그인), Google Play App Signing(앱 서명) 사용 중

## 3. 증상
### 관찰된 현상
- `./build.sh release` 로 빌드한 산출물을 **Google Play Store에 올려 설치하면 앱 무결성 검증이 실패**.
- 동일 산출물을 **`adb install`로 직접 설치하면 정상 통과**.

### 재현 절차
1. `./build.sh release` 로 release 빌드 생성
2. (A) Play Console(내부 테스트 등)에 업로드 → 기기에 설치 → 앱 실행 → 무결성 검사 실패
3. (B) 같은 APK를 `adb install` → 앱 실행 → 무결성 검사 통과

### 에러 메시지 / 로그 (원문)
```text
(앱 동작상 무결성 검증 실패 분기로 진입 — alert "앱 무결성 검증에 실패..." 후 종료)
logcat 확인 시:
W IntegrityChecker: Current signature hash: A22867E706B879DD54CCD687898ED480F15564F6A63C0AFA8C757DE0B67A4C32
```

## 4. 확인한 내용
- 무결성 검사 핵심 지점: `plugins/cordova-plugin-integrity/src/android/IntegrityChecker.java`
  - L30 `getPackageInfo(pkg, GET_SIGNING_CERTIFICATES)` → L32 `signingInfo.getApkContentsSigners()[0]` 의 SHA-256 을
    상수 `EXPECTED_SIGNATURE_HASH`(L17)와 비교.
  - 기대값(원래): `5A9469841062D7EFAB7F08DEDF0C85766584FEA2CEDF1120F4441D6A126DA83E` = **로컬 업로드 키**(`keyStore/hanchartapplite_release_key.jks`) 해시.
- `adb logcat -s IntegrityChecker` 로 Play 배포본의 실제 해시 확인:
  - `A22867E706B879DD54CCD687898ED480F15564F6A63C0AFA8C757DE0B67A4C32` → 기대값과 **불일치**.
- 결과 해석: Play 배포본은 내 업로드 키가 아니라 **다른 키로 서명**되어 있다 → Google Play App Signing의 **앱 서명 키**로 재서명된 것.

## 5. 원인 분석
- **확정 원인:** Google Play App Signing이 켜져 있어, 업로드한 AAB/APK를 Google이 **자기 "앱 서명 키"로 다시 서명**해서 사용자에게 배포한다.
  무결성 검사가 보는 `getApkContentsSigners()`는 **사용자 기기에 설치된 최종 APK의 서명자** = Google 앱 서명 키 → 기대값(업로드 키 해시)과 달라 실패.
  반면 `adb install`은 **재서명 안 거친 로컬 업로드 키 서명본**을 그대로 설치하므로 일치 → 통과.
- 즉 "release인데 adb는 되고 Play만 안 됨"의 뿌리는 debug/release 차이가 아니라 **업로드 키 ≠ 앱 서명 키**.

## 6. 조치 내역
- 단일 기대값 비교를 **허용 목록(allowlist) 비교**로 변경. 업로드 키 + Play 앱 서명 키 둘 다 통과.
- 수정 파일 (둘 다 — 플러그인 원본 + 컴파일 대상 복사본):
  - `plugins/cordova-plugin-integrity/src/android/IntegrityChecker.java`
  - `platforms/android/app/src/main/java/com/trustnhope/hanchartapplite/IntegrityChecker.java`
- 핵심 diff:
  ```java
  // before
  private static final String EXPECTED_SIGNATURE_HASH = "5A9469841062D7EFAB7F08DEDF0C85766584FEA2CEDF1120F4441D6A126DA83E";
  ...
  return EXPECTED_SIGNATURE_HASH.equals(currentHash);

  // after
  private static final String[] EXPECTED_SIGNATURE_HASHES = {
      "5A9469841062D7EFAB7F08DEDF0C85766584FEA2CEDF1120F4441D6A126DA83E", // 업로드 키 (로컬/adb)
      "A22867E706B879DD54CCD687898ED480F15564F6A63C0AFA8C757DE0B67A4C32"  // Play 앱 서명 키
  };
  ...
  for (String expected : EXPECTED_SIGNATURE_HASHES) {
      if (expected.equals(currentHash)) return true;
  }
  return false;
  ```
- 조치 결과: adb install 본(업로드 키)과 Play 배포본(앱 서명 키) 양쪽 모두 통과하도록 정정. (Play 트랙 재배포 검증은 다음 빌드에서 확인 예정)

## 7. 최종 상태
- 해결 여부: 코드 수정 완료 (허용 목록화).
- 남은 확인: `./build.sh release` 후 Play 내부 테스트 트랙 재업로드 → 실제 통과 확인.

## 8. 학습 포인트
- 핵심 개념: **Play App Signing = 업로드 키와 앱 서명 키의 분리.** 사용자가 받는 APK 서명자는 내 업로드 키가 아니라 Google의 앱 서명 키다.
- 이번에 배운 점: 서명 해시 기반 무결성 검사는 Play App Signing 환경에서 **반드시 앱 서명 키 해시를 허용 목록에 포함**해야 한다.
- 다음에 같은 증상이면 확인할 순서:
  1) `adb logcat -s IntegrityChecker` 로 배포본 실제 해시 확인
  2) Play Console > 앱 무결성 > 앱 서명 키 인증서 SHA-256 과 대조
  3) 코드 허용 목록에 그 값이 있는지 확인
- 👉 별도 학습 노트: [[2026-06-09-play-app-signing-upload-vs-app-signing-key]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 앱/새 키 등록 시 Play Console 앱 서명 키 SHA-256을 허용 목록에 추가했는가
  - [ ] 무결성 검사 기대값을 단일 상수가 아닌 **목록**으로 유지하는가 (업로드 키 + 앱 서명 키)
  - [ ] `plugins/` 원본과 `platforms/android` 복사본 둘 다 반영했는가
- 추가하면 좋을 로깅: 실패 분기에서 현재 해시 + 기대 목록을 함께 로그(이미 현재 해시는 L83에 출력 중).

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: "adb install은 되는데 Play(또는 특정 트랙) 설치본만 무결성 실패"
- 확인 절차: logcat의 `Current signature hash` ↔ Play Console 앱 서명 키 SHA-256 비교
- 조치 절차: 불일치 해시를 `EXPECTED_SIGNATURE_HASHES` 에 추가 → 재빌드/재배포
- 정상 확인 기준: adb 설치본 / Play 설치본 모두 무결성 통과

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "혹시 현재 buils.sh 로 진행할 때 구그 ㄹ플레이 스토어 올리면 무결성 검증이 실패해 ㅎ그런데 adb install apk fmf gkaus wkf emfdjrkwlsmsep antms answpdlswl dkfdk/" (= "build.sh로 빌드해 구글 플레이 스토어에 올리면 무결성 검증이 실패해. 그런데 adb install apk를 하면 잘 들어가지는데 무슨 문제인지 알아?")
- "그런데 adb install apk 를 하게 되면 드러가지는데 무슨 문제인지 알아?"
- "adb logcat -s IntegrityChecker 이거 어떻게 ㅎ ㅏ는건지 step by step???"
- "그럼 배포하기전에 계속 이와 같이 해야하는거야?"

### Claude가 되물은 확인 질문 + 답
- (별도 확인 질문 없음 — 코드/로그 직접 확인으로 원인 규명. Play 앱 서명 키 해시는 사용자가 logcat 값으로 제공: `A22867E7...4C32`)

## 12. 한 줄 결론
> Play App Signing은 업로드본을 Google 앱 서명 키로 재서명하므로, 서명 해시 무결성 검사는 업로드 키와 앱 서명 키 해시를 **둘 다 허용 목록에** 넣어야 한다.
