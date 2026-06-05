# Flutter 무선 ADB 기기 설치 — flutter install의 두 함정

> 한 줄 요약: 무선(mDNS) ADB 기기에 Flutter 앱 깔 땐 `flutter install` 대신 `adb -s <전체serial> install -r <apk>` 가 안전하다.

## 핵심
- 무선 ADB(Android 11+) 기기의 serial은 `adb-<HW>._adb-tls-connect._tcp` 형태로 **점(.)을 포함**한다.
- `flutter install -d <id>` 는 이 serial을 첫 점에서 잘라 `adb -s` 에 넘겨, `adb: device '<잘린id>' not found` 로 실패한다.
- 게다가 `flutter install` 의 **기본 빌드 모드는 release** 라, debug만 빌드해 둔 상태면 `app-release.apk does not exist` 로 또 실패한다.
- 둘 다 adb 직접 설치로 한 번에 회피된다. flutter는 산출물 빌드까지만 맡기고, 설치는 adb에 맡기는 게 단순하고 확실.

## 언제 / 왜 쓰나
- iPad/태블릿을 USB 없이 무선 디버깅으로 붙여 테스트할 때.
- 여러 기기가 동시에 붙어 있어 특정 기기만 골라 깔아야 할 때(`-s` 로 지정).
- debug 빌드를 빠르게 재설치하며 검증할 때(`-r` reinstall).

## 예시 (명령)
```bash
# 0. 최초 1회 무선 연결 (기기 '무선 디버깅' 화면 값 사용)
adb pair  <IP>:<페어링포트> <6자리코드>
adb connect <IP>:<연결포트>

# 1. 기기 전체 serial 확인 (점 포함 전체를 그대로 써야 함)
adb devices -l
#  → adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp  model:SM_X510  transport_id:8

# 2. 빌드 (debug)
flutter build apk --debug      # → build/app/outputs/flutter-apk/app-debug.apk

# 3. 설치 (adb 직접, 전체 serial)
adb -s adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp \
    install -r build/app/outputs/flutter-apk/app-debug.apk
#  → Performing Streamed Install / Success
```

## 흔한 함정 / 헷갈리는 점
- **점 잘림**: `flutter run/install` 의 무선 serial 파싱 버그. adb 직접 호출하면 전체 문자열이 보존돼 회피.
- **release 기본**: `flutter install` 은 산출물을 빌드하지 않고 release apk가 있다고 가정. debug 깔려면 adb로.
- **transport_id 는 휘발성**: `-t 8` 같은 transport_id는 재연결 시 바뀐다. 스크립트엔 **전체 serial** 을 쓸 것.
- **기기 구분**: `adb devices -l` 의 `model:` 필드가 기기 보고값(신뢰 가능). 마케팅 이름은 모델코드→이름 추정일 뿐.
- 페어링 포트와 연결 포트는 **다르다**(무선 디버깅 메인 화면 vs 페어링 코드 화면).

## 질문 기록
### 내가 던진 질문 (원문)
- "내가 할 수 있게 명령어를 전달해줘"
- "1. 모델을 어떻게 알았어?"

### Claude가 되물은 확인 질문 + 답
- Q: 새로 연결하려는 기기는 어떤 종류/방식? / A: 무선 ADB (Android)

## 관련
- 발단이 된 이슈: [[2026-06-05-flutter-build-install-wireless-adb]]
- 참고: Android 무선 디버깅(adb pair/connect), `flutter install` 기본 release 모드

---
- 생성일: 2026-06-05
- 마지막 갱신: 2026-06-05
