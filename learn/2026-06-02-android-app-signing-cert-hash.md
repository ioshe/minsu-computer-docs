# 안드로이드 앱 서명: debug/release 키스토어와 인증서 SHA-256 해시

> 한 줄 요약: 안드로이드 앱은 keystore(.jks) 안의 **인증서**로 서명되고, 무결성 검사가 비교하는 "앱 서명 해시"는 **그 인증서의 SHA-256 지문**(keytool의 `SHA256:`와 같은 값)이다. debug 빌드는 SDK 자동 생성 `~/.android/debug.keystore`로, release는 내 `.jks`로 서명되어 해시가 서로 다르다.

## 핵심
- 키스토어(`.jks`) 안에는 **개인키 + 공개키 인증서(X.509)**가 들어 있다. APK는 이 키로 서명된다.
- "앱 서명 해시"라고 부르는 값은 **jks 파일의 해시도, APK 내용의 해시도 아니고**, 서명에 쓰인 **인증서(certificate)의 DER 바이트를 SHA-256** 한 값이다.
- 그래서 `keytool -list -v`가 보여주는 `SHA256:` 지문과 **같은 값**이다.
- **빌드 타입마다 다른 키 → 다른 해시:**
  - **release**: 내가 만든 `keyStore/vegassolution_release_key.jks` → `8B717936...4C42F58B`
  - **debug**: 안드로이드 SDK가 자동 생성한 `~/.android/debug.keystore`(alias `androiddebugkey`, storepass `android`) → `c13b3a8d...0fb00edf`
- debug 해시는 **코드/스크립트 어디에도 박혀 있지 않다.** debug 빌드를 AGP가 debug 키로 자동 서명한 결과를, 런타임에 `PackageManager`로 꺼내 계산한 것일 뿐.

## 언제 / 왜 쓰나
- 앱 무결성/위변조 검사: "이 앱이 정식 키로 서명됐나"를 서명 인증서 지문으로 검증.
- Google/Firebase/카카오/네이버 등 SDK 등록: SHA-1/SHA-256 지문 등록 시 debug/release 둘 다 등록해야 함(키가 다르므로).
- "release는 되는데 debug만 안 됨" 류 문제 진단.

## 예시 (코드·명령)
```bash
# release 키스토어 인증서 지문
keytool -list -v -keystore keyStore/vegassolution_release_key.jks -storepass '!10dnjfdo' | grep SHA256
#   SHA256: 8B:71:79:36:...:4C:42:F5:8B   (= 코드의 EXPECTED_SIGNATURE_HASH)

# debug 키스토어 인증서 지문
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android | grep SHA256
#   SHA256: C1:3B:3A:8D:...:B0:0E:DF

# APK가 실제로 어떤 키로 서명됐는지
apksigner verify --print-certs app-debug.apk | grep SHA-256
#   certificate SHA-256 digest: c13b3a8d...0fb00edf
```
```java
// 런타임에 "현재 서명 인증서"를 꺼내 SHA-256 (IntegrityChecker.java)
PackageInfo pi = pm.getPackageInfo(pkg, PackageManager.GET_SIGNING_CERTIFICATES); // L30
Signature sig = pi.signingInfo.getApkContentsSigners()[0];                        // L32
byte[] digest = MessageDigest.getInstance("SHA-256").digest(sig.toByteArray());   // L52-53
```

## 흔한 함정 / 헷갈리는 점
- **"해시 = jks 파일 해시"가 아니다.** jks 안의 **인증서**를 해시한 것. (그래서 keytool `SHA256:`와 동일)
- **debug 해시는 비밀이 아니고 박혀 있지도 않다.** 누구의 머신이든 `~/.android/debug.keystore`는 보통 같은 alias/비번(android)이지만 인증서 자체는 머신마다 생성되어 값이 다를 수 있다 → "내 debug 해시"는 내 기기 기준.
- **build.sh는 해시를 주입하지 않는다.** 무결성 플러그인 코드만 주입하고, 기대 해시는 소스 상수(`IntegrityChecker.java:17`)에 하드코딩, 현재 해시는 런타임 계산.
- debug↔release 상호 설치 시 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`도 같은 뿌리(서명 인증서 다름).

## 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "위 내용에서 debug 해쉬는 내가 알기론 없는데 build.sh 에서 뭐 집어 넣어?? 그리고 해당 해쉬의 값 꺼내 쓰는게 내가 아는 jks 파일 이랑 같은거야? 아니면 뭐가 다른건가? 해당 해쉬값은 어떤 해쉬 값인거니?"

### Claude가 되물은 확인 질문 + 답
- (이 주제에서 별도 확인 질문 없음 — 직접 keytool/apksigner로 값을 계산해 답함)

## 관련
- 발단이 된 이슈: [[2026-06-02-debug-integrity-blocks-chrome-inspect]], [[2026-06-02-install-failed-update-incompatible]]
- 연결 노트: [[2026-06-02-android-context-info]] (PackageManager로 꺼내는 정보), [[2026-06-02-client-integrity-check-limits]]
- 참고 링크: keytool / apksigner (Android SDK build-tools)

---
- 생성일: 2026-06-02
- 마지막 갱신: 2026-06-02
