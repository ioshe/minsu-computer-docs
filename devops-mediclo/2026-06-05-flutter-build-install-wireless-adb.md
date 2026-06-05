# 2026-06-05 · Flutter debug APK 빌드 후 무선 ADB 새 기기(SM X510) 설치

## 1. 개요
- 날짜: 2026-06-05
- 프로젝트 / 화면 / 모듈: devops-mediclo (Mediclo, Flutter 태블릿 EMR 앱) / 앱 전체 빌드·배포
- 이슈 유형: 환경 / 빌드·배포 (설치 실패 후 우회 성공)
- 계층: 빌드·배포 / 네이티브·플러그인(adb)
- 최종 상태: 해결 (adb 직접 설치로 우회)
- 중요도: 보통

## 2. 환경
- OS / 기기 / 웹뷰: 개발 PC Ubuntu 24.04.4 LTS (6.17.0-29-generic) / 타깃 태블릿 2대
  - 신규: `SM_X510` (Android 16, API 36, android-arm64) — 무선 ADB
  - 기존: `SM_T500` (Android 12, API 31, android-arm64) — 무선 ADB
- Flutter / Dart: Flutter 3.38.5 (stable, f6ff1529fd) / Dart 3.10.4
- 브랜치 / 커밋: develop / 9d6e173 (working tree에 진료실 관련 미커밋 변경 다수)
- 관련 의존성: 43개 패키지가 제약조건상 최신 아님(경고만, 빌드엔 영향 없음)

## 3. 증상
### 관찰된 현상
- `flutter build apk --debug` 는 정상 성공 (`✓ Built build/app/outputs/flutter-apk/app-debug.apk`, Gradle assembleDebug 46.5s).
- 새 기기 SM X510에 `flutter install -d <wireless-id>` 시도 → 두 가지로 실패.

### 재현 절차
1. Android 11+ 기기를 무선 디버깅으로 `adb pair` → `adb connect` 하여 연결.
2. `flutter install -d adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp` 실행.

### 에러 메시지 / 로그 (원문)
```text
$ flutter install -d adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp
Error 1 retrieving device properties for SM X510:
adb: device 'adb-R54XA008HCF-PkV9a4' not found

Installing app-release.apk to SM X510...
Uninstalling old version...
"build/app/outputs/flutter-apk/app-release.apk" does not exist.
Install failed
```

## 4. 확인한 내용
- `adb devices -l` 로 두 기기 모두 `device` 상태 정상 확인:
  - `adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp  model:SM_X510  transport_id:8`
  - `adb-R9TRA0DX21B-OC5fhk._adb-tls-connect._tcp  model:SM_T500  transport_id:6`
- 빌드 산출물은 `app-debug.apk` 만 존재(`build/app/outputs/flutter-apk/`), `app-release.apk` 는 없음.
- 결과 해석:
  - 에러 1줄 `device 'adb-R54XA008HCF-PkV9a4' not found` → flutter가 무선 기기 ID를 **첫 점(`.`)에서 잘라** `adb -s` 에 넘김. 실제 serial은 `..._adb-tls-connect._tcp` 까지가 전체.
  - `app-release.apk does not exist` → `flutter install` 의 기본 빌드 모드는 **release**. 우리는 debug만 빌드했으므로 산출물 불일치.

## 5. 원인 분석
- **확정 원인 (2가지 독립):**
  1. **기기 ID 점 잘림** — `flutter install`/`flutter run` 의 device id 파싱이 mDNS 무선 serial(`adb-XXXX._adb-tls-connect._tcp`)의 점 뒤를 truncate. → adb가 기기를 못 찾음.
  2. **빌드 모드 불일치** — `flutter install` 기본값 release ↔ 빌드한 산출물 debug. `app-release.apk` 부재로 즉시 실패.
- 가능성 있는 원인: 없음(위 2개로 로그 전부 설명됨).

## 6. 조치 내역
- 우회책(workaround) — adb로 **전체 serial 지정 + 이미 빌드된 debug APK** 직접 설치:
  ```bash
  adb -s adb-R54XA008HCF-PkV9a4._adb-tls-connect._tcp \
      install -r build/app/outputs/flutter-apk/app-debug.apk
  ```
  - `-s <전체 serial>`: 점 잘림 회피 (전체 문자열 그대로 전달).
  - `-r`: 기존 설치 유지하며 재설치(reinstall).
  - `install` 대상으로 debug APK 명시 → 모드 불일치 회피.
- 조치 결과:
  ```text
  Performing Streamed Install
  Success
  ```

## 7. 최종 상태
- 해결 여부: 해결 — SM X510에 debug 빌드 설치 성공.
- 남은 문제 / 추가 확인:
  - release 설치가 필요하면 `flutter build apk --release` 먼저 빌드해야 함.
  - transport_id(`-t 8`)는 재연결 시 바뀔 수 있으므로 스크립트엔 **전체 serial** 사용 권장.

## 8. 학습 포인트
- 핵심 개념: 무선 ADB 기기에 Flutter 설치 시 (1) device id 점 잘림, (2) install 기본 release 모드 — 둘 다 adb 직접 설치로 우회 가능.
- 이번에 배운 점: `flutter install`은 빌드 산출물을 알아서 안 만들고 **이미 빌드된 release apk를 가정**. debug를 깔려면 adb로 직접.
- 다음에 같은 증상이면 확인할 순서: 1) `adb devices -l` 로 전체 serial·상태 확인 2) 빌드한 모드(debug/release)와 `flutter install` 기본 모드 일치 여부 3) 무선 기기면 flutter 경유 대신 adb 직접 설치
- 👉 별도 학습 노트: [[2026-06-05-flutter-wireless-adb-install]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 무선 기기 설치는 `adb -s <전체serial> install -r <apk>` 패턴 사용
  - [ ] `flutter install` 쓸 거면 동일 모드로 먼저 빌드(`--release` 기본)
  - [ ] 기기 구분은 `model:` 필드로 (SM_X510=신규, SM_T500=기존)
- 추가하면 좋을 것: 설치용 셸 스크립트(`scripts/install_debug.sh`)로 serial·apk 경로 고정.
- 자동화 후보: device serial을 인자로 받아 `flutter build apk --debug && adb -s $1 install -r ...` 하는 래퍼.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `adb: device '<잘린id>' not found` 또는 `app-release.apk does not exist`.
- 확인 절차: `adb devices -l` → 기기 `device` 상태 & 전체 serial 확보 → `ls build/app/outputs/flutter-apk/`.
- 조치 절차: `adb -s <전체serial> install -r build/app/outputs/flutter-apk/app-debug.apk`.
- 정상 확인 기준: `Performing Streamed Install` 다음 `Success` 출력.

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "FLUTTER build 및 앱설치"
- "아 새로운 기기로 연결하려고하는데"
- "내가 할 수 있게 명령어를 전달해줘"
- "devices id 가 어떤게 내가 원하는거지 어떻게  tab"
- "1. 모델을 어떻게 알았어?"

### Claude가 되물은 확인 질문 + 답
- Q: 새로 연결하려는 기기는 어떤 종류이고, 어떤 방식으로 연결하시나요?
  - A: 무선 ADB (Android)
- Q: (모델을 어떻게 알았나에 대한 답) — `adb devices -l` 의 `model:` 필드가 기기 보고값이며, "갤럭시 탭 S9 FE/A7" 마케팅명은 모델코드 기반 추정.
  - A: (설명 제공, 사용자 추가 반론 없음)

## 12. 한 줄 결론
> 무선 ADB 기기 + Flutter debug 설치는 `flutter install`(점 잘림 + release 기본)을 피하고 `adb -s <전체serial> install -r app-debug.apk` 로 직접 깐다.
