# Play App Signing — 업로드 키 vs 앱 서명 키, 누가 무엇으로 서명하나

> 한 줄 요약: Google Play App Signing이 켜져 있으면 키가 **둘**로 분리된다. 나는 **업로드 키**로 서명해 올리고, Google이 그걸 떼어내 **앱 서명 키**로 다시 서명해 사용자에게 배포한다. 그래서 사용자 기기에 설치된 APK의 서명 해시는 내 업로드 키가 아니라 **Google 앱 서명 키 해시**이고, `adb install`로 직접 깐 것만 업로드 키 해시다.

## 핵심
- **업로드 키 (Upload key)** = 내 로컬 키스토어(`keyStore/hanchartapplite_release_key.jks`). 내가 AAB/APK를 서명해 Play Console에 **업로드할 때** 신원 증명용으로만 쓰인다. 사용자에게 가지 않는다.
- **앱 서명 키 (App signing key)** = Google이 보관·관리하는 키. Play가 내 업로드본을 검증한 뒤 **이 키로 다시 서명**해서 실제 배포본을 만든다. 사용자 기기에 깔리는 APK의 진짜 서명자.
- 무결성 검사가 보는 `signingInfo.getApkContentsSigners()` 는 **"지금 이 기기에 설치된 APK를 서명한 자"** → 배포 경로에 따라 값이 달라진다:
  - **`adb install`(로컬 산출물 직접 설치)** → 재서명 안 거침 → **업로드 키** 해시
  - **Play Store 설치(내부테스트 포함)** → Google 재서명 → **앱 서명 키** 해시
- 그래서 단일 기대값으로 비교하면 둘 중 하나는 반드시 깨진다. **둘 다 허용 목록에** 넣어야 양쪽 다 통과.
- **앱 서명 키는 앱당 영구 고정**이다. Play App Signing 등록 시 한 번 정해지면 앱을 삭제하지 않는 한 안 바뀐다 → 허용 목록에 **한 번만** 넣으면 끝, 배포마다 갱신할 필요 없다.

## 이번 케이스의 두 해시
| 키 | 출처 | SHA-256 | 언제 보이나 |
|---|---|---|---|
| 업로드 키 | 로컬 `hanchartapplite_release_key.jks` | `5A946984...126DA83E` | `adb install` 설치본 |
| 앱 서명 키 | Google Play App Signing | `A22867E7...B67A4C32` | Play Store 배포본 |

## 흐름 (release 빌드 → 사용자)
```
[내 PC]  ./build.sh release
            │  업로드 키(.jks)로 서명
            ▼
        AAB/APK  ──(업로드)──►  [Play Console]
                                    │  업로드 키 서명 검증 후 제거
                                    │  앱 서명 키로 재서명
                                    ▼
                                배포본 APK  ──►  [사용자 기기]
                                                   getApkContentsSigners() = 앱 서명 키 해시

  vs.  adb install (Play 안 거침) → 업로드 키 서명 그대로 → 업로드 키 해시
```

## 언제 / 왜 쓰나
- 서명 해시 기반 무결성/위변조 검사를 하는 앱을 **Play App Signing으로 배포**할 때.
- "release 빌드인데 adb는 되고 Play 설치본만 무결성/SDK 인증 실패" 진단.
- Firebase/카카오/네이버 등 SHA 지문 등록도 동일 함정: 업로드 키뿐 아니라 **앱 서명 키 지문도 등록**해야 Play 배포본에서 동작.

## 앱 서명 키 SHA-256 얻는 두 경로
1. **Play Console** → 해당 앱 → 테스트 및 출시 > 설정 > **앱 무결성(App signing)** → "앱 서명 키 인증서" SHA-256.
2. **logcat** (이미 코드가 현재 해시를 찍는 경우): Play 배포본 실행 후 `adb logcat -s IntegrityChecker` → `Current signature hash:` 값.
   - 두 경로 값이 같아야 정상(교차 검증). 콘솔 값은 콜론 포함 대문자(`AB:CD:..`), 코드 비교 시엔 콜론 제거.

## 흔한 함정 / 헷갈리는 점
- **"release 키로 서명했으니 release 해시일 것"** ❌ — Play를 거치면 사용자 기기의 서명자는 release(업로드) 키가 아니라 **앱 서명 키**다.
- debug/release 차이(→ [[2026-06-02-android-app-signing-cert-hash]])와 **다른 축**의 문제다. 여긴 같은 release 산출물인데 **배포 경로(adb vs Play)** 로 갈린다.
- 허용 목록에 업로드 키만 두면 adb 테스트는 통과해 "고쳤다"고 착각 → Play 올리면 또 실패. 반드시 양쪽 다.
- `plugins/` 원본과 `platforms/android` 복사본 두 군데 다 고쳐야 한다(빌드 시 원본이 복사본을 덮지만, 직접 컴파일 대상은 복사본).

## 질문 기록
### 내가 던진 질문 (원문)
- "그리고 기존에 있던 키랑 이번에 있던 키랑 인증방식이 어떻게 다르고 애 ㅗ다르고 어떤 흐름으로 흘러가는지 정리해봐야 할 듯"

### Claude가 되물은 확인 질문 + 답
- (이 주제에서 별도 확인 질문 없음 — 세션 중 코드·logcat으로 확인한 사실로 정리)

## 관련
- 발단이 된 이슈: [[2026-06-09-play-store-integrity-resign]]
- 연결 노트: [[2026-06-02-android-app-signing-cert-hash]] (debug/release 키스토어·인증서 해시 — 같은 "서명 해시"지만 debug↔release 축), [[2026-06-02-client-integrity-check-limits]], [[2026-06-02-cordova-native-vs-webview-integrity]]
- 참고 링크: Play Console > 앱 무결성(App signing)

---
- 생성일: 2026-06-09
- 마지막 갱신: 2026-06-09
