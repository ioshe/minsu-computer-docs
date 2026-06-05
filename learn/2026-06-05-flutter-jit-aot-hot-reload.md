# Flutter 빌드 모드(JIT/AOT)와 hot reload 동작 원리

> 한 줄 요약: debug=JIT, release=AOT. JIT는 hot reload의 *필요조건*일 뿐이고, 실제 hot reload는 PC의 `flutter run/attach`가 변경분을 재컴파일해 기기 VM에 주입해야 일어난다.

## 핵심
- Flutter는 **빌드 모드 플래그(`--debug`/`--release`/`--profile`) 자체가 컴파일 방식 스위치**다. 따로 켜는 설정은 없다.
  - `--debug` → **JIT**(Just-In-Time): Dart 코드를 기기에서 실행 중 컴파일. hot reload·assert·디버그 배너 가능. 느리고 APK 큼.
  - `--release` → **AOT**(Ahead-Of-Time): 빌드 시점에 네이티브 ARM 기계어로 미리 컴파일. 빠르고 작음. hot reload 불가.
  - `--profile` → AOT + 프로파일링(성능 측정용).
- **JIT면 hot reload가 자동으로 되는 게 아니다.** JIT는 "새 코드를 받아들일 수 있는 상태"일 뿐, 새 코드를 *밀어넣어 줄 주체*가 따로 있어야 한다.
- 그 주체가 PC의 `flutter run`(또는 실행 중 앱에 붙는 `flutter attach`)다. 단독 설치된 debug 앱(아이콘 탭 실행)은 JIT라도 PC가 안 붙어 있어 hot reload가 안 일어난다.

## 언제 / 왜 쓰나
- 앱을 깔아서 **그냥 실행/시연/QA** 만 할 때 → `flutter build apk` + `adb install`(또는 `flutter install`)로 충분. `flutter run` 불필요.
- 코드 고치며 **즉시 반영(hot reload)·디버깅·로그** 가 필요할 때 → `flutter run -d <기기>` (빌드+설치+실행+attach 올인원) 또는 이미 떠 있는 앱에 `flutter attach`.

## 예시 (코드·명령)
```text
# hot reload 실제 동작 순서
[PC]  파일 변경 감지(watch) → 바뀐 Dart만 kernel(.dill)로 재컴파일
        │  (컴파일은 기기가 아니라 PC가 한다)
        ▼  VM Service 채널로 전송
[기기] 실행 중 Dart VM이 새 kernel 주입 → 위젯 트리 rebuild

# 단순 실행 (hot reload 불필요)
flutter build apk --debug
adb -s <전체serial> install -r build/app/outputs/flutter-apk/app-debug.apk
# → 기기에서 아이콘 탭 실행 (단독 동작)

# 개발용 (hot reload 필요)
flutter run -d <기기id>          # 빌드+설치+실행+attach
flutter attach -d <기기id>        # 이미 실행 중인 debug 앱에 붙기
```

| 상황 | JIT? | hot reload |
|---|---|---|
| `flutter run -d <기기>` | O | O (PC가 watch+push) |
| 실행 중 앱에 `flutter attach` | O | O |
| 아이콘 탭으로 debug APK 단독 실행 | O | ✗ (PC 미연결) |
| release APK 실행 | ✗ (AOT) | ✗ (애초에 불가) |

## 흔한 함정 / 헷갈리는 점
- **"JIT = hot reload 자동"은 오해.** JIT는 필요조건, PC 연결(`flutter run/attach`)이 충분조건. 둘 다 있어야 함.
- **컴파일 주체 착각.** hot reload의 재컴파일은 *기기*가 아니라 *PC*가 한다. 기기 VM은 받은 kernel을 주입만 한다.
- **debug 앱 단독 실행 ≠ hot reload.** 아이콘으로 띄운 debug 앱은 JIT로 돌지만 hot reload 채널이 안 열려 있다.
- **release는 hot reload 자체가 불가.** AOT라 런타임에 코드 교체가 안 됨.
- hot reload(상태 유지, 위젯 rebuild) vs hot restart(상태 초기화, 앱 재시작)는 또 다름 — 둘 다 `flutter run` 연결 상태에서만.

## 질문 기록
### 내가 던진 질문 (원문)
- "이거 just in time으로 되는거는 어떻게 뭐 다른 설정이 따ㅓ로 있나?"
- "jjit 컴파일을 하면 hot reload 가 되는거 아니었나"
- "그러면 fluter run 을 안해도 된다는거지?"

### Claude가 되물은 확인 질문 + 답
- Q: "just in time"이 JIT 컴파일이 아니라 다른 걸(hot reload 즉시 반영 등) 가리킨 거면 말씀해 달라 / A: (별도 정정 없음 — JIT 맥락으로 이어서 질문)

## 관련
- 발단이 된 이슈: [[2026-06-05-flutter-build-install-wireless-adb]]
- 관련 노트: [[2026-06-05-flutter-wireless-adb-install]]
- 참고: Flutter build modes(debug/profile/release), Dart VM Service, hot reload

---
- 생성일: 2026-06-05
- 마지막 갱신: 2026-06-05
