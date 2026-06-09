# learn — 학습 노트 주제별 인덱스

이슈를 고치다 건진 **재사용 가능한 개념**을 주제별로 모은 곳. (프로젝트·날짜 무관)
이슈의 시간순 뷰는 [`months/`](../months/)에, 여긴 **주제로 찾는** 인덱스다.
각 노트의 파일명 앞 날짜는 *생성일*(고정)이며, 내용은 같은 주제면 제자리 갱신된다.

## 안드로이드 · 하이브리드(Cordova) 앱 — 보안·서명·웹뷰

| 생성일 | 주제 |
|---|---|
| 06-02 | [안드로이드 앱 서명: debug/release 키스토어와 인증서 SHA-256 해시](./2026-06-02-android-app-signing-cert-hash.md) |
| 06-02 | [Android Context로 얻는 정보 + 빌드 타입별로 바뀌는 값](./2026-06-02-android-context-info.md) |
| 06-02 | [클라이언트 무결성 검사의 한계 & BuildConfig.DEBUG 우회의 안전성](./2026-06-02-client-integrity-check-limits.md) |
| 06-02 | [Cordova 하이브리드 앱의 이중 보안 가드 — 네이티브(Activity) vs WebView(JS)](./2026-06-02-cordova-native-vs-webview-integrity.md) |
| 06-02 | [Cordova JS ↔ Java 브리지 동작 원리 (cordova.exec)](./2026-06-02-cordova-js-java-bridge.md) |
| 06-02 | [chrome://inspect로 WebView 원격 디버깅이 뜨는 기준 (FLAG_DEBUGGABLE)](./2026-06-02-chrome-inspect-webview-debugging.md) |
| 06-09 | [Play App Signing — 업로드 키 vs 앱 서명 키, 누가 무엇으로 서명하나](./2026-06-09-play-app-signing-upload-vs-app-signing-key.md) |

## 빌드 · 런타임 · 배포

| 생성일 | 주제 |
|---|---|
| 06-04 | [AGP 8.x — BuildConfig는 opt-in (buildFeatures.buildConfig)](./2026-06-04-agp8-buildconfig-feature.md) |
| 06-05 | [Flutter 빌드 모드(JIT/AOT)와 hot reload 동작 원리](./2026-06-05-flutter-jit-aot-hot-reload.md) |
| 06-05 | [Flutter 무선 ADB 기기 설치 — flutter install의 두 함정](./2026-06-05-flutter-wireless-adb-install.md) |
| 06-08 | [Flutter 렌더링 파이프라인 — Dart 코드부터 GPU 픽셀까지 (Skia / Impeller)](./2026-06-08-flutter-rendering-pipeline-skia-impeller.md) |

## Dart 언어 · 모델 · 직렬화

| 생성일 | 주제 |
|---|---|
| 06-06 | [Dart `DateTime.tryParse` 함정 — 구분자 없는 `yyyyMMdd` 는 파싱 실패](./2026-06-06-dart-datetime-tryparse-yyyymmdd.md) |
| 06-06 | [Dart `implements` 시그니처 드리프트 → 런타임 NoSuchMethodError](./2026-06-06-dart-implements-signature-drift-nosuchmethod.md) |
| 06-08 | [freezed + flutter/foundation 전체 import → DTO 마다 Diagnosticable mixin 생성](./2026-06-08-freezed-foundation-import-diagnosticable.md) |
| 06-08 | [json_serializable 타입 불일치 + catch(_) = silent 데이터 손실](./2026-06-08-json-serializable-type-mismatch-silent-loss.md) |

## 상태관리 · 생명주기 · 데이터 흐름 (Riverpod)

| 생성일 | 주제 |
|---|---|
| 06-05 | [Riverpod `ref` / `context` 는 async gap(await)을 넘기지 마라](./2026-06-05-riverpod-ref-context-async-gap.md) |
| 06-06 | [Flutter `dispose()` — 위젯 생명주기 정리 메서드](./2026-06-06-flutter-dispose.md) |
| 06-06 | [Flutter(Riverpod) 진료실 데이터 흐름 구조 — 처음 보는 사람을 위한 긴 설명](./2026-06-06-flutter-진료실-데이터흐름-구조.md) |
| 06-06 | [Riverpod 기초 — 상태관리·Provider·Notifier·build·ref·최적화](./2026-06-06-riverpod-basics-입문.md) |

## Flutter UI · 위젯 · 작성 패턴

| 생성일 | 주제 |
|---|---|
| 06-06 | [Flutter — 커서(캐럿) 위치 텍스트 삽입 & 포커스 보정](./2026-06-06-flutter-cursor-text-insertion.md) |
| 06-06 | [Flutter 고정폭 위젯에서 텍스트가 잘리는 두 원인 (한글 softWrap + 시스템 textScaler)](./2026-06-06-flutter-fixed-width-text-truncation.md) |
| 06-06 | [Flutter 작성 시트 자동저장 패턴 — Mixin 템플릿메서드 + Debounce + 로컬 드래프트](./2026-06-06-flutter-autosave-sheet-mixin-debounce.md) |

## 네트워크 · API · 서비스 레이어

| 생성일 | 주제 |
|---|---|
| 06-05 | [Dio 인증·baseUrl을 인터셉터에 위임하기 (Thin Service 패턴)](./2026-06-05-dio-interceptor-auth-delegation.md) |
| 06-05 | [Flutter 서비스 레이어 결과/에러 타입 전략 (Either vs ApiResult vs throw)](./2026-06-05-flutter-service-result-error-type-strategy.md) |
| 06-06 | [생성/수정 응답이 "식별자만" 줄 때 — 클라이언트 모델 재구성](./2026-06-06-mutation-response-identifier-only.md) |
