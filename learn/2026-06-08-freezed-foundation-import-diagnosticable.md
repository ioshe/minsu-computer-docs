# freezed + flutter/foundation 전체 import → DTO 마다 Diagnosticable mixin 생성

> 한 줄 요약: 순수 DTO 파일에 `import 'package:flutter/foundation.dart';` 를 통째로 넣으면 freezed 가 모든 클래스에 `DiagnosticableTreeMixin`/`debugFillProperties` 를 자동 생성해 생성 코드가 불필요하게 팽창한다. `show` 로 필요한 심볼만 노출하면 막힌다.

## 핵심
- freezed 는 라이브러리 스코프에 `Diagnosticable`(= `flutter/foundation` 또는 `flutter/widgets`)가 **import 되어 있으면**, 생성하는 각 `_$XxxImpl` 에 `with DiagnosticableTreeMixin` + `debugFillProperties(...)` + `toString({DiagnosticLevel ...})` 를 자동으로 끼워 넣는다.
- DTO 파일은 보통 foundation 이 필요 없는데, **로깅 때문에 `kDebugMode`/`debugPrint` 하나 쓰려고 foundation 전체를 import** 하면 이 mixin 이 딸려와 freezed 생성물이 클래스마다 수십 줄씩 불어난다(순수 churn).
- 해결: `import 'package:flutter/foundation.dart' show kDebugMode, debugPrint;` — `Diagnosticable` 이 스코프에 안 들어와 freezed 가 mixin 을 안 만든다.
- 주의 타이밍: import 를 바꿔도 freezed 를 **재생성(build_runner)** 하기 전엔 생성 파일이 안 바뀐다. 그래서 "import 추가 커밋"과 "freezed 변화"가 다른 시점에 드러날 수 있다(나중에 재생성할 때 갑자기 큰 diff 로 튀어나옴).

## 언제 / 왜 쓰나
- freezed DTO/모델 파일에서 디버그 로깅(`kDebugMode`, `debugPrint`)이나 `compute`, `ValueListenable` 등 foundation 심볼이 필요할 때.
- 생성된 `*.freezed.dart` 가 갑자기 `DiagnosticableTreeMixin`/`debugFillProperties` 로 200줄씩 늘어났을 때 원인 추적.

## 예시 (코드·명령)
```dart
// ❌ DTO 마다 DiagnosticableTreeMixin/debugFillProperties 생성됨
import 'package:flutter/foundation.dart';

// ✅ 필요한 심볼만 — mixin 생성 안 됨
import 'package:flutter/foundation.dart' show kDebugMode, debugPrint;
```
확인:
```bash
dart run build_runner build -d
grep -c "DiagnosticableTreeMixin" lib/.../xxx.freezed.dart   # 0 이면 깔끔
```

## 흔한 함정 / 헷갈리는 점
- `kDebugMode`, `debugPrint` 둘 다 `flutter/foundation` 소속이라 "foundation 을 안 쓸" 수는 없다 — 핵심은 **`show` 로 범위만 좁히는 것**.
- mixin 자체는 기능상 무해(디버그 출력 강화)하지만, 생성 코드 비대 + diff 노이즈 때문에 순수 DTO 에선 보통 원치 않음.
- git revert 로 생성물을 되돌린 뒤 build_runner 가 `wrote 0 outputs` 로 스킵하면, 캐시(asset graph) 때문이다 → 해당 `*.g.dart`/`*.freezed.dart` 를 **직접 삭제 후 재생성**하면 강제된다.

## 질문 기록
### 내가 던진 질문 (원문)
- "나중에 ㄱrelease 때는 로그 안나오겠지?" (→ `debugPrint` 단독은 release 에서도 실행됨. `if (kDebugMode)` 로 감싸야 트리쉐이킹으로 제거. 이 과정에서 foundation 전체 import 가 mixin 을 부르는 함정을 만남)

### Claude가 되물은 확인 질문 + 답
- (없음)

## 관련
- 발단이 된 이슈: [[2026-06-08-progress-v2-disease-order-silent-parse-loss]]
- 함께 본 함정: [[2026-06-08-json-serializable-type-mismatch-silent-loss]]
- 참고 링크: —

---
- 생성일: 2026-06-08
- 마지막 갱신: 2026-06-08
