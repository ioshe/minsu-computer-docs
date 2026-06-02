# 2026-06-02 · adb install 시 `no devices/emulators found` (USB 레벨 미인식)

## 1. 개요
- 날짜: 2026-06-02
- 프로젝트 / 화면 / 모듈: vegasAppLite-app / APK 기기 설치
- 이슈 유형: 환경
- 계층: 디바이스·OS (USB/adb 인식)
- 최종 상태: 해결 (기기 인식됨 — 이후 별도 이슈로 이어짐)
- 중요도: 보통

## 2. 환경
- OS / 기기 / 브라우저·웹뷰: Linux 데스크톱 (Ubuntu 계열, 커널 6.17.0-29-generic)
- 앱 버전 / 빌드: vegasapp-debug-20260602_120439.apk
- 브랜치 / 커밋: feature/isms
- 관련 의존성·버전: adb 1.0.41 (platform-tools 37.0.0), `/home/minsu/Android/Sdk/platform-tools/adb`

## 3. 증상
### 관찰된 현상
- `adb install <apk>` 실행 시 `adb: no devices/emulators found`
- 사용자는 "선(USB 케이블)은 연결한 상태"라고 함

### 재현 절차
1. Linux 데스크톱에 안드로이드 폰을 USB로 연결
2. `adb install /home/minsu/src/vegasAppLite-app/output/vegasapp-debug-20260602_120439.apk` 실행

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
adb install .../vegasapp-debug-20260602_120439.apk
adb: no devices/emulators found
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점:
  - `adb devices -l` → `List of devices attached` (비어 있음)
  - `lsusb` → USB 허브(Genesys Logic), RGB LED 컨트롤러, 루트 허브만. **안드로이드 기기 항목이 아예 없음**
- 결과 해석: adb가 못 찾는 게 아니라, **운영체제 USB 레벨에서 폰 자체가 안 잡힘**. 즉 adb 이전 단계(케이블/USB 모드/연결) 문제.

## 5. 원인 분석
- **확정 원인:** USB 레벨에서 기기 미인식. `lsusb`에 폰이 안 나타나므로 adb가 디바이스를 찾을 수 없음.
- **가능성 있는 원인:** ① 충전 전용 케이블(데이터 미전송) ② USB 연결 모드가 "충전 전용"(MTP/PTP 아님) ③ USB 디버깅 미활성화 ④ USB 허브 경유로 인한 인식 불안정
- 근거: `lsusb` 결과에 안드로이드 기기가 전혀 없음 → 소프트웨어(adb) 아닌 물리/연결 계층.

## 6. 조치 내역
- 수행한 조치 (확인 순서 안내):
  1. **케이블 교체** (충전 전용 의심) + **USB 허브 거치지 말고 PC 직결**
  2. 폰 알림창에서 **USB 모드 → 파일 전송(MTP)/PTP** 로 변경
  3. **개발자 옵션 → USB 디버깅 ON**
  4. 매 단계 후 `lsusb`로 폰이 새 항목으로 뜨는지 확인
  5. 폰은 `lsusb`엔 뜨는데 `adb devices`엔 안 보이면 → `adb kill-server && adb start-server && adb devices`, 폰 화면의 "USB 디버깅 허용" 팝업 승인
- 조치 결과: 이후 사용자가 `adb install`을 다시 실행했을 때 기기는 인식됨 (다음 에러 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`로 진행). 즉 인식 문제는 해소됨.

## 7. 최종 상태
- 해결 여부: 해결 (기기 인식됨)
- 남은 문제 / 추가 확인 필요: 인식 후 서명 불일치 설치 실패 → [[2026-06-02-install-failed-update-incompatible]] 로 이어짐

## 8. 학습 포인트
- 핵심 개념: `adb: no devices/emulators found`는 **adb 문제가 아닐 수 있다.** 먼저 `lsusb`로 OS가 기기를 보는지부터 확인하라.
- 이번에 배운 점: adb 디버깅의 진단 순서는 **물리(lsusb) → 연결 모드 → USB 디버깅 → adb 서버/권한** 순으로 내려간다.
- 다음에 같은 증상이면 확인할 순서: 1) `lsusb`에 폰이 있나 2) 케이블/포트(허브 배제) 3) USB 모드 MTP 4) USB 디버깅 ON 5) `adb kill-server`+권한 팝업

## 9. 재발 방지
- 체크리스트:
  - [ ] 데이터 전송 가능한 케이블 사용 (충전 전용 X)
  - [ ] 허브 거치지 말고 PC 직결
  - [ ] 폰 USB 모드 = 파일 전송, USB 디버깅 ON
- 추가하면 좋을 테스트 / 가드 / 로깅: 설치 스크립트 앞단에 `adb get-state`/`lsusb` 사전 체크 한 줄
- 자동화 후보: `adb wait-for-device` 후 설치

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `adb: no devices/emulators found`
- 확인 절차: `lsusb` → 폰 있나? 없으면 물리/케이블/모드 / 있는데 `adb devices` 비면 권한(udev)·서버
- 조치 절차: 케이블/포트 교체 → USB 모드 MTP → USB 디버깅 ON → `adb kill-server && adb start-server` → 폰 팝업 허용
- 정상 확인 기준: `adb devices`에 기기 시리얼이 `device` 상태로 표시

## 11. 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "리눅스 데크스톱에서 다음과 같은데 선은 연결한상태야" (adb: no devices/emulators found 에러와 함께)

### Claude가 되물은 확인 질문 + 답
- (별도의 예/아니오 확인 질문 없음 — 진단 순서를 안내하고 "케이블 교체 + 직결 + USB 모드 변경 후 `lsusb` 결과를 알려달라"고 요청)
  - A: 사용자가 이후 `adb install`을 재실행 → 기기 인식됨(다음 에러로 진행)

## 12. 한 줄 결론
> `no devices/emulators found`는 adb 탓부터 의심하지 말고 `lsusb`로 OS가 기기를 보는지부터 확인하라 — 이번엔 USB 레벨 미인식이었다.
