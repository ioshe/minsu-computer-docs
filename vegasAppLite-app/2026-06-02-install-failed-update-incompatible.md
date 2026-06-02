# 2026-06-02 · `INSTALL_FAILED_UPDATE_INCOMPATIBLE` (서명 불일치로 설치 실패)

## 1. 개요
- 날짜: 2026-06-02
- 프로젝트 / 화면 / 모듈: vegasAppLite-app / APK 기기 설치
- 이슈 유형: 빌드 실패(설치/서명)
- 계층: 빌드·배포 (APK 서명/패키징)
- 최종 상태: 해결 (원인 규명 + 해결책 제시)
- 중요도: 보통

## 2. 환경
- OS / 기기 / 브라우저·웹뷰: Linux 데스크톱 + 안드로이드 실기기
- 앱 버전 / 빌드: vegasapp-debug-20260602_120439.apk (debug, applicationId `com.trustnhope.vegas`)
- 브랜치 / 커밋: feature/isms

## 3. 증상
### 관찰된 현상
- 기기 인식 후 `adb install` 시 설치 실패. 기존에 설치된 같은 패키지(`com.trustnhope.vegas`)와 **서명이 다르다**는 메시지.

### 재현 절차
1. 폰에 `com.trustnhope.vegas`가 (이전 서명으로) 이미 설치돼 있음
2. 다른 키로 서명된 APK를 `adb install`로 덮어쓰기 시도

### 에러 메시지 / 스택 트레이스 / 로그 (원문)
```text
Performing Streamed Install
adb: failed to install .../vegasapp-debug-20260602_120439.apk:
Failure [INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.trustnhope.vegas
signatures do not match previously installed version; ignoring!]
```

## 4. 확인한 내용
- 실행한 명령어 / 확인 지점: 에러 메시지 자체 분석 + 빌드 서명 구조 확인
- 결과 해석: 안드로이드는 **이미 설치된 앱과 새 APK의 서명 키가 다르면 업데이트(덮어쓰기)를 거부**한다. 기존엔 release 키로 설치돼 있었는데 이번엔 debug 키로 빌드 → 서명 불일치.

## 5. 원인 분석
- **확정 원인:** 동일 패키지의 기존 설치본과 새 APK의 **서명 인증서가 다름**. (release 키로 설치된 앱 위에 debug 키 APK 설치 시도)
- 근거: 에러 문구 `signatures do not match previously installed version`. (서명 키가 debug/release로 갈리는 상세는 [[2026-06-02-debug-integrity-blocks-chrome-inspect]] 및 [[2026-06-02-android-app-signing-cert-hash]] 참고)

## 6. 조치 내역
- 임시 우회책(workaround): 기존 앱 삭제 후 재설치
  ```bash
  adb uninstall com.trustnhope.vegas
  adb install <새 apk>
  ```
  ⚠️ 기존 앱의 로컬 데이터(로그인/설정)도 함께 삭제됨.
- 근본 수정(root fix): **기존 설치본과 동일한 키로 서명**해서 빌드 (예: release 설치본이면 release 키로 빌드 → 서명 일치 → 데이터 보존하며 덮어쓰기 가능). `adb install -r` 은 서명이 같을 때만 의미 있음.
- 조치 결과: 사용자는 데이터 삭제 없이 가려고 release 빌드 방향으로 진행 (이후 무결성/디버깅 이슈로 연결).

## 7. 최종 상태
- 해결 여부: 해결 (원인·해결책 확정)
- 남은 문제 / 추가 확인 필요: debug로 설치/디버깅하려는 본래 목적은 무결성 검사 문제로 이어짐 → [[2026-06-02-debug-integrity-blocks-chrome-inspect]]

## 8. 학습 포인트
- 핵심 개념: 안드로이드는 **앱 업데이트 시 서명 동일성**을 강제한다. debug 키와 release 키는 서로 다른 인증서라 상호 덮어쓰기가 안 된다.
- 이번에 배운 점: debug ↔ release 전환 설치 시엔 거의 항상 이 에러 → uninstall이 가장 빠른 우회.
- 다음에 같은 증상이면 확인할 순서: 1) 기존 설치본이 어떤 키로 서명됐나 2) 데이터 보존 필요? → 같은 키로 서명 / 아니면 uninstall

## 9. 재발 방지
- 체크리스트:
  - [ ] 같은 기기에 debug/release를 섞어 설치하지 않기 (테스트 빌드 타입 통일)
  - [ ] 데이터 보존이 필요하면 동일 키 서명 유지
- 추가하면 좋을 테스트 / 가드 / 로깅: 설치 스크립트에서 실패 시 자동으로 `adb uninstall` 후 재시도(데이터 삭제 동의 플래그 하에)
- 자동화 후보: `adb install -r || (adb uninstall <pkg> && adb install)` (데이터 삭제 주의)

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: `INSTALL_FAILED_UPDATE_INCOMPATIBLE ... signatures do not match`
- 확인 절차: 새 APK 서명(`apksigner verify --print-certs`)과 기존 설치본 서명 비교
- 조치 절차: 동일 키 서명 후 재설치 / 또는 `adb uninstall <pkg>` 후 설치
- 정상 확인 기준: 설치 성공, 앱 정상 실행

## 11. 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- (명시적 질문 문장 없이 에러 로그를 붙여넣음) "adb install ... Failure [INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.trustnhope.vegas signatures do not match previously installed version; ignoring!]"

### Claude가 되물은 확인 질문 + 답
- Q: 기존 앱을 uninstall 하고 재설치할까요? (기존 로컬 데이터가 삭제됩니다) / 데이터 보존이 필요하면 release 키로 동일 서명해야 합니다. 어느 쪽으로 할까요?
  - A: 사용자가 uninstall을 택하지 않고, "properties가 잘 세팅됐는지 확인" → release 빌드/무결성 방향으로 진행 (직접 예/아니오 답 대신 다른 경로 선택)

## 12. 한 줄 결론
> 같은 패키지라도 서명 키가 다르면 못 덮어쓴다 — debug↔release 전환 설치는 uninstall 또는 동일 키 서명으로 푼다.
