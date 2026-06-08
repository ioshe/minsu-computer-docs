# Flutter 렌더링 파이프라인 — Dart 코드부터 GPU 픽셀까지 (Skia / Impeller)

> 한 줄 요약: 우리가 짠 Dart 위젯이 화면 픽셀로 찍히기까지는 **Framework(Dart) → Engine(C++) → 2D 그래픽 엔진(Skia/Impeller) → GPU** 의 단방향 파이프라인을 거치며, Skia→Impeller 교체의 핵심은 "셰이더를 런타임이 아니라 빌드 타임에 미리 컴파일해 Shader Jank를 없앤 것"이다.

## 핵심

전체 그림을 세 레이어로 끊어서 본다.

1. **Framework (Dart)** — 우리가 짜는 위젯. UI의 *설계도*를 만든다.
2. **Engine (C++)** — 설계도를 받아 "무엇을 그릴지" 명령 목록(DisplayList)으로 만든다.
3. **그래픽 엔진 (Skia / Impeller)** — 그 목록을 GPU가 알아듣는 신호(셰이더·드로우콜)로 번역해 픽셀로 굽는다.
4. (그 아래 **Embedder** — Android/iOS/Windows 등 플랫폼별로 surface·스레드·이벤트 루프를 붙여주는 접착층.)

비유: Framework/Engine은 *"파란 사각형 그리고 위에 글씨 써"* 라는 **설계도**, Skia/Impeller는 그 설계도를 받아 **실제 페인트칠을 하는 일꾼**. 일꾼이 구형 장비(Skia, 런타임 셰이더 컴파일)에서 신형 장비(Impeller, 빌드타임 컴파일)로 바뀌면서 "처음 보는 그림 그릴 때 버벅대던 버릇(Shader Jank)"이 사라진 게 핵심 사건이다.

---

### 1) 세 그루의 트리 — Widget / Element / RenderObject

Flutter UI는 동시에 존재하는 **3개의 병렬 트리**로 돌아간다. 흔한 오해가 "위젯 트리 하나만 있다"인데, 실제로는 역할이 갈린다.

| 트리 | 수명 | 역할 |
|---|---|---|
| **Widget** | 매우 짧음(불변·매 빌드마다 새로 생성) | UI의 *설정값/설계도*. `Container(color: blue)` 는 그냥 설정 객체일 뿐, 아무것도 안 그린다. |
| **Element** | 김(상태 유지) | Widget과 RenderObject를 잇는 *중개자*. 트리의 위치·식별자·상태를 들고 있다. `BuildContext` 가 곧 Element. |
| **RenderObject** | 김(상태 유지) | 실제 **레이아웃·페인팅**을 담당하는 무거운 객체. 비싸서 재생성 대신 재사용한다. |

- `setState` → 해당 Element가 dirty 표시 → 다음 프레임에 `build()` 재호출 → **새 Widget**과 **기존 Element**를 비교(diff).
- 이 비교를 빠르게 만드는 게 **`Widget.canUpdate`**: `runtimeType` 과 `key` 가 같으면 → Element를 **재사용**하고 RenderObject에 새 설정값만 갈아끼운다(`updateRenderObject`). 다르면 → Element/RenderObject를 버리고 새로 만든다.
- **이것이 위젯을 매 프레임 통째로 새로 만들어도 안 느린 이유**: 비싼 건 RenderObject인데 그건 안 버리고 재사용하기 때문. Widget은 가볍고 일회용이다.
- `const` 위젯이 성능에 좋은 이유도 여기: 같은 인스턴스라 `canUpdate` 단계에서 빠르게 걸러져 rebuild가 잘려나간다.

> 면접 포인트: "왜 build가 자주 불려도 괜찮은가?" → Widget은 불변·경량 설정 객체이고, 실제 비용은 Element가 보존하는 RenderObject에 있으며 diff로 최소 변경만 반영하기 때문.

### 2) 한 프레임의 생애 — 파이프라인 단계

Vsync 신호가 오면 `RendererBinding.drawFrame()` 이 다음 단계를 **순서대로** 돈다. 각 단계는 dirty로 표시된 노드만 처리한다.

```
Vsync ──▶ ① Animate (애니메이션 틱, Ticker)
       ──▶ ② Build  (dirty Element들 rebuild → Widget diff)
       ──▶ ③ Layout (RenderObject: 부모→자식 constraints 내려보내고, 자식→부모 size 올림)
       ──▶ ④ Paint  (RenderObject.paint() → Canvas 호출 → DisplayList 기록)
       ──▶ ⑤ Compositing (레이어 트리 → Scene)
       ──▶ ⑥ Rasterize (Skia/Impeller가 Scene을 GPU 드로우콜로 변환) ◀── 여기서 GPU
```

- **Layout 규칙**: "Constraints go down, Sizes go up, Parent sets position." 부모가 자식에게 *허용 범위(min/max width·height)* 를 내려주고, 자식이 그 안에서 자기 크기를 정해 올려보내면, 부모가 위치를 잡는다. 한 번의 트리 순회로 끝나서 O(n)에 가깝다(웹의 reflow보다 단순).
- **Paint 단계의 핵심 산출물 = DisplayList**: `RenderObject.paint()` 안에서 `canvas.drawRect(...)` 를 호출하면 **즉시 GPU에 그리는 게 아니라**, "사각형 그려라" 라는 **명령을 리스트에 기록**만 한다. 이 직렬화된 그리기 명령 목록이 **DisplayList**(예전 SkPicture/SkCanvas 기록과 같은 개념). → *나중에* 별도 스레드에서 한 방에 재생(replay)된다.
- **왜 명령을 기록만 하나?** UI 스레드(Dart)와 Raster 스레드를 분리하기 위해서. UI 스레드는 "할 일 목록(DisplayList)"만 빨리 적고 넘기고, 실제 GPU 노가다는 Raster 스레드가 한다. 이 분리가 60/120fps 유지의 토대.

### 3) Dart UI ↔ C++ Engine 브릿지 — 제어권이 넘어가는 지점

- Framework가 만든 **Layer 트리**는 `SceneBuilder` 로 **Scene** 객체가 된다(`dart:ui` 라이브러리).
- `dart:ui` 의 `Canvas`, `Picture`, `Scene`, `SceneBuilder` 같은 클래스들은 **Dart 쪽 껍데기**고, 실제 구현은 **C++ Engine**에 있다. 둘을 잇는 게 **`@Native` / FFI 스타일 바인딩**(과거엔 `native` 키워드/`tonic`).
- 결정적 호출: **`PlatformDispatcher.render(Scene)`** → 이 한 줄에서 **제어권이 Dart에서 C++ Engine으로 넘어간다.**
- Engine은 받은 Scene/DisplayList를 **Raster 스레드**에서 Skia 또는 Impeller에 넘겨 GPU 드로우콜로 변환 → Embedder가 만든 surface에 그려 화면에 표시.

> 데이터 흐름 요약: `setState` → (Dart) Build/Layout/Paint → DisplayList + Layer 트리 → `SceneBuilder.build()` → `render(Scene)` **[Dart→C++ 경계]** → (C++ Raster 스레드) Skia/Impeller → GPU.

---

### 4) Skia vs Impeller — 무엇이, 왜 바뀌었나

**Skia**: 구글의 범용 2D 그래픽 라이브러리(Chrome, Android에도 쓰임). 강력하지만 Flutter 입장에서 약점이 하나 있었다 → **Shader Jank(셰이더 쟁크)**.

#### Shader Jank가 생기는 메커니즘 (Skia)

- GPU에 뭔가 그리려면 **셰이더 프로그램**(GLSL 등)이 필요하다. Skia는 "지금 그릴 도형/효과(그라데이션·블러·특정 블렌드모드 등)"에 맞는 셰이더를 **그 효과가 화면에 처음 등장하는 그 순간(런타임)에 생성·컴파일**한다.
- 셰이더 컴파일은 비싸다(드라이버가 GPU 기계어로 변환). 이게 **프레임 도중**에 일어나면 그 프레임이 16ms를 넘겨 **버벅임(jank)** 으로 보인다.
- 특히 **앱을 처음 켜거나 새 화면/새 애니메이션이 처음 등장할 때** 와르르 몰려서 첫인상이 끊긴다.
- Skia 시절 우회책: **SkSL warm-up**(미리 셰이더 캐시를 구워두기) — 번거롭고 기기/드라이버마다 달라 완벽하지 않았다.

#### Impeller의 해법 — "런타임 컴파일을 없앤다"

Impeller는 Flutter 전용으로 새로 만든 렌더러다. 핵심 아이디어: **셰이더를 런타임에 만들지 말고, 빌드 타임에 다 만들어 둔다.**

1. **빌드 타임 셰이더 컴파일 (`impellerc`)**
   - Impeller가 쓰는 셰이더는 **개수가 정해져 있고 작다**(Flutter가 그릴 수 있는 효과 집합이 한정적이므로). 이걸 미리 GLSL/Metal Shading Language 등으로 작성해 둔다.
   - **`impellerc`** 컴파일러가 **앱 빌드 시점에** 이 셰이더들을 각 백엔드(Metal/Vulkan/GLES)용 **중간 표현/바이너리**로 미리 컴파일한다. (내부적으로 SPIR-V 등을 거쳐 백엔드별로 변환)
   - 결과: 앱이 실제로 그릴 때 **컴파일이 0번** 일어난다. 이미 구워진 걸 꺼내 쓰기만 함 → **Shader Jank 원천 제거.**

2. **정적 파이프라인 상태 객체 (Static Pipeline State Objects, PSO)**
   - Metal/Vulkan 같은 모던 API는 "셰이더 + 블렌드 상태 + 정점 레이아웃 + 래스터 상태"를 묶은 **PSO(파이프라인 상태 객체)** 를 만들어 GPU에 등록하는 구조다. 이 PSO 생성도 비싸다.
   - Skia(`GrContext` 기반)는 그릴 때 상황에 맞춰 동적으로 상태를 조합 → 런타임 비용·예측 불가.
   - Impeller는 필요한 PSO 조합이 **유한·고정**임을 알고 있으므로 **미리(또는 앱 시작 시 한 번) 만들어 캐싱**한다. 프레임 도중엔 이미 만든 PSO를 바인딩만 → 끊김 없음.
   - 정리: **Skia = 동적·런타임 파이프라인 / Impeller = 정적·예측가능 파이프라인.**

3. **모던 그래픽 API의 멀티스레딩 활용**
   - Impeller는 처음부터 **Metal(iOS), Vulkan(Android)** 같은 모던·저수준 API를 1순위 백엔드로 설계(OpenGL은 폴백).
   - 이들 API는 **커맨드 버퍼를 여러 스레드에서 병렬로 기록**할 수 있다. Impeller는 이걸 활용해 GPU 작업 준비를 분산 → CPU 병목 완화.
   - 또한 그리기 자원(버텍스 버퍼 등)을 미리 확보/재사용하는 구조라 **프레임당 할당이 적다** → 뒤에서 말할 GC/메모리 압력 감소로 이어진다.

> 주의(정확성): "Impeller가 GC 부담을 줄인다"는 표현은 흔하지만 정밀하게는 — Impeller가 줄이는 건 주로 **C++ 네이티브 측의 런타임 할당/컴파일 비용**과 **프레임 스파이크**다. Dart의 GC와 직접 연결된 건 아니다. (아래 5번 참고.) 면접에서 "Impeller가 Dart GC를 줄였다"고 단정하면 과장이다.

---

### 5) Dart Managed Heap ↔ C++ Native Memory 연동

화면을 그릴 때 메모리는 두 세계에 걸쳐 있다.

- **Dart 객체(Managed Heap)**: `Image`, `Picture`, 위젯 래퍼 등. Dart VM의 GC(세대별 GC)가 관리.
- **네이티브 메모리(Native Heap)**: 실제 **GPU 텍스처·버텍스 버퍼·디코딩된 비트맵** 등. Skia/Impeller(C++)가 `malloc`/GPU 할당으로 관리. **GC가 모른다.**

문제: Dart의 작은 `ui.Image` 객체 하나가 네이티브 쪽 **수 MB 텍스처**를 붙들고 있을 수 있다. Dart GC는 "이 객체 작네" 하고 늦게 수거할 수 있어, 네이티브 메모리가 안 풀리고 쌓일 위험이 있다.

해결 메커니즘:

1. **`NativeFinalizer` / Finalizer**
   - Dart 객체가 GC로 수거될 때, 그에 묶인 **네이티브 리소스 해제 콜백**을 호출하도록 등록(`dart:ffi` 의 `NativeFinalizer`). → Dart 객체가 죽으면 C++ 텍스처도 `free` 된다.
   - 단, **GC 시점은 비결정적**이라 "언제 풀릴지"는 보장 안 됨.
2. **명시적 `dispose()` (권장)**
   - 그래서 `ui.Image`, `Picture`, `ImageDescriptor` 등은 **`dispose()`** 를 제공한다. GC를 기다리지 말고 **즉시** 네이티브 메모리를 풀라는 신호.
   - 거대한 리스트뷰·이미지 갤러리에서 메모리 폭증을 막으려면 안 쓰는 이미지의 `dispose()`/캐시 관리가 필수. (관련: 위젯 측 [[2026-06-06-flutter-dispose]])
3. **참조 카운팅(C++ 내부)**: Skia의 `sk_sp`(smart pointer) / Impeller의 RAII 기반 리소스 관리로, 네이티브 측에서도 마지막 참조가 사라지면 자동 해제.

**Skia → Impeller 메모리 측면 개선**:
- Impeller는 파이프라인/셰이더를 미리 만들어 **장기 캐싱**하고, 프레임마다 새 자원을 할당하기보다 **풀(pool)에서 재사용**하는 설계 → **프레임당 네이티브 할당·해제 스파이크가 적다.**
- 결과적으로 메모리 사용이 더 **평탄(predictable)** 해지고, 갑작스런 할당으로 인한 jank가 준다. (단, 미리 만들어 두는 게 있어 *초기/유휴 메모리 footprint* 는 케이스에 따라 비슷하거나 더 들 수도 있음 — "무조건 적다"가 아니라 "더 예측가능하다"가 정확.)

---

## 언제 / 왜 쓰나 (이 지식이 실전에서 쓰이는 순간)

- **첫 화면/첫 애니메이션 진입 시 버벅임** 디버깅 → Shader Jank 의심 → Impeller 사용 여부 확인. (Flutter 3.x부터 iOS 기본 Impeller, Android 점진 기본화)
- **DevTools 성능 탭에서 "Shader compilation" 표시된 빨간 프레임** → Skia라면 warm-up, 가능하면 Impeller 전환.
- **이미지 많은 화면에서 메모리 OOM/누수** → `ui.Image.dispose()`, `ImageCache` 크기 조정, `cacheWidth/cacheHeight` 로 디코딩 크기 제한.
- **면접/아키텍처 설명** — "Dart 코드가 픽셀까지 가는 과정" 질문에 위 6단계 파이프라인 + 3트리 + Dart↔C++ 경계(`render(Scene)`)를 짚으면 깊이가 드러난다.

## 예시 (개념 확인용 코드)

```dart
// 1) Widget은 그냥 설정값 — 아무것도 그리지 않는다.
const blue = Container(color: Colors.blue); // 불변·경량. 매 빌드마다 새로 만들어도 싸다.

// 2) 직접 캔버스에 그리는 자리 = RenderObject/Painter (여기서 DisplayList가 기록된다)
class _BoxPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    // 이 호출은 "즉시 GPU에 그리기"가 아니라 DisplayList에 명령을 '기록'하는 것.
    canvas.drawRect(Offset.zero & size, Paint()..color = Colors.blue);
  }
  @override
  bool shouldRepaint(covariant CustomPainter old) => false;
}

// 3) 네이티브 텍스처를 붙든 Dart 객체는 다 쓰면 명시적으로 해제 (GC를 기다리지 않는다)
final ui.Image img = await loadImage();
canvas.drawImage(img, Offset.zero, Paint());
img.dispose(); // ← 네이티브 메모리 즉시 반환. 안 하면 GC 시점까지 텍스처가 남는다.
```

```bash
# Impeller 사용/비활성 토글로 Shader Jank 원인 확인하기 (디버깅용)
flutter run --enable-impeller        # 강제 활성화
flutter run --no-enable-impeller     # Skia로 폴백해 증상 비교
```

## 흔한 함정 / 헷갈리는 점

- **"위젯 트리만 있다"** ❌ → Widget/Element/RenderObject **3트리**. 비용은 RenderObject에 있고 Widget은 일회용.
- **`canvas.drawXxx()` 가 즉시 그린다고 착각** ❌ → **DisplayList에 기록만** 하고, 실제 래스터화는 Raster 스레드에서 나중에. (그래서 UI 스레드가 안 막힌다.)
- **"Impeller가 Dart GC를 줄인다"** ⚠️ → 정밀하게는 **네이티브 측 런타임 컴파일·할당 스파이크**를 줄이는 것. Dart GC와 직접 연결 아님. 과장 주의.
- **"Impeller는 항상 메모리를 덜 쓴다"** ⚠️ → "더 **예측가능**하다"가 정확. 미리 굽는 자원 때문에 초기 footprint는 비슷할 수 있음.
- **Shader Jank vs 일반 Jank 혼동** → Shader Jank는 **셰이더 컴파일**로 인한 *첫 등장 시* 끊김. 무거운 build/layout으로 인한 jank와 원인·해법이 다르다(후자는 Impeller로 안 풀림).
- **`dispose()` 안 부르고 GC만 믿기** → 거대 리스트/이미지에서 네이티브 메모리 누적·OOM. Finalizer는 안전망일 뿐 타이밍 보장이 없다.
- **`impellerc` 를 우리가 직접 돌린다고 오해** ❌ → Flutter 빌드 툴체인이 엔진 셰이더에 대해 내부적으로 수행. 앱 개발자가 손댈 일은 보통 없다(커스텀 프래그먼트 셰이더 `shaders:` 쓸 때만 간접 관여).

## 질문 기록
> 이 항목은 항상 채운다(원칙 8).

### 내가 던진 질문 (원문)
- "내가 Flutter 앱에서 Dart 코드로 'Container(color: Colors.blue)'를 생성했을 때, 이 코드가 화면의 실제 파이프라인을 거쳐 Skia 또는 Impeller 엔진을 통해 GPU에서 픽셀로 렌더링되기까지의 전체 과정을 단계별로 설명해줘. ... 1. Widgets -> Elements -> RenderObjects 트리의 변환 과정 2. 레이아웃(Layout)과 페인팅(Painting) 단계에서 생성되는 디스플레이 리스트(DisplayList)의 개념 3. Dart UI 라이브러리와 C++ Engine 간의 브릿지 역할"
- "Flutter의 새로운 엔진인 Impeller가 기존 Skia의 'Shader Jank(셰이더 쟁크)'를 해결한 구체적인 아키텍처적 메커니즘을 설명해줘. ... 1. Impeller가 빌드 타임(AOT)에 'Impellerc'라는 컴파일러를 사용해서 GLSL/MSL 같은 셰이더 소스코드를 어떻게 미리 처리하는지? 2. Skia의 GrContext 기반 파이프라인과 Impeller의 정적 파이프라인(Static Pipeline State Objects) 구조의 차이점 3. Impeller가 Metal이나 Vulkan 같은 모던 그래픽 API의 '병렬 처리(Multi-threading)' 기능을 어떻게 활용하여 가비지 컬렉션(GC) 부담을 줄이는지?"
- "Flutter 앱에서 UI가 그려질 때 Dart가 관리하는 메모리(Managed Heap)와 C++ 엔진(Skia/Impeller)이 관리하는 네이티브 메모리(Native Memory)가 어떻게 상호작용하는지 딥하게 설명해줘. ... 1. Dart 객체(예: UI 위젯 이미지 래퍼)가 소멸될 때 C++ 네이티브 단의 텍스처(Texture)나 그래픽 버퍼 메모리가 해제되는 메커니즘(Pointer, Finalizer 등) 2. Skia에서 Impeller로 바뀌면서 네이티브 메모리 관리나 할당(Allocation) 효율성 측면에서 어떤 개선이 있었는지 구체적으로 알려줘."

### Claude가 되물은 확인 질문 + 답
- (없음 — 잘 정립된 개념 정리라 추가 정보 수집 없이 작성. 단, 본문에 "흔한 과장/부정확 표현"을 ⚠️로 명시해 사실 경계를 표시함.)

## 관련
- 발단이 된 이슈: (없음 — 자발적 학습)
- 관련 노트: [[2026-06-05-flutter-jit-aot-hot-reload]] (AOT/JIT·Hot Reload 맥락), [[2026-06-06-flutter-dispose]] (dispose로 네이티브 자원 해제), [[2026-06-06-flutter-진료실-데이터흐름-구조]]
- 참고 링크: Flutter 공식 — flutter.dev/go/impeller, "Flutter architectural overview", "Inside Flutter" 문서

---
- 생성일: 2026-06-08
- 마지막 갱신: 2026-06-08
