# Riverpod 기초 — 상태관리·Provider·Notifier·build·ref·최적화

> 한 줄 요약: Riverpod은 "데이터를 위젯 바깥의 공용 창고(Provider)에 두고, watch로 구독하면 값이 바뀔 때 그 위젯만 자동으로 다시 그려주는" Flutter 상태관리 라이브러리다.

## 핵심

- Flutter의 대전제: **화면 = 데이터의 함수** (`UI = f(데이터)`). 데이터가 바뀌면 Flutter가 화면을 다시 그린다. "데이터 바뀐 걸 화면이 어떻게 알지?"를 푸는 게 **상태관리**.
- **상태(state)** = 시간이 지나며 변하고, 변하면 화면도 같이 변해야 하는 데이터. (좋아요 수, 로그인 여부, 메모 목록 등)
- Riverpod의 아이디어: **데이터를 위젯 안에 두지 말고, 위젯 바깥 "공용 창고"에 둬라.** 그러면 중간 위젯에 손에서 손으로 넘길(props drilling) 필요 없이 누구든 직접 꺼내 보고, 창고가 바뀌면 보던 위젯이 자동 갱신된다.
- 용어 3개:
  - **Provider** = 데이터를 담는 공용 창고. (단, 변수 자체는 데이터가 아니라 "창고를 찾는 주소/열쇠"다 — 아래 최적화 ① 참고)
  - **watch(구독)** = "이 창고 보고 있을게, 바뀌면 나도 다시 그려줘".
  - **read(읽기)** = "지금 값 한 번만 줘" (계속 보지는 않음, 주로 버튼 콜백).
- **단순 Provider** = 값만 든 상자. **Notifier** = 값 + 그 값을 바꾸는 메서드가 달린 똑똑한 상자. Notifier 안에서 **`state = 새값`** 을 하면 그 창고를 구독한 위젯이 자동 리빌드된다.

## 언제 / 왜 쓰나

- 여러 화면이 같은 데이터를 공유해야 할 때 (로그인 정보, 목록 등).
- 깊이 중첩된 위젯에 데이터를 전달해야 할 때(props drilling 회피).
- "데이터 바뀌면 화면 자동 갱신"이 필요한 거의 모든 경우. Mediclo는 `flutter_riverpod` + `riverpod_generator`(코드 생성)를 표준으로 쓴다.

## 예시 (코드)

가장 단순한 창고(고정값):
```dart
final counterProvider = Provider<int>((ref) => 42);
// 위젯에서
final count = ref.watch(counterProvider); // 42
```

변하는 데이터 = Notifier (코드 생성 방식):
```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;          // 처음 값(자동 1회 실행)
  void increment() => state = state + 1; // 값 바꾸면 구독 위젯 자동 갱신
}
```

비동기 데이터(서버/지연) = `Future` 반환 → 로딩/에러/성공 3상태:
```dart
@riverpod
class CurrentJusojungList extends _$CurrentJusojungList {
  @override
  Future<List<Jusojung>> build(String scheduleId) async {
    await Future.delayed(Duration(milliseconds: 350)); // 네트워크 흉내
    return _sortJusojungList(_items);
  }
  Future<void> togglePin(String id) async {
    _items = /* 수정 */;
    state = AsyncData(...); // 구독 위젯 자동 갱신
  }
}
```

화면에서 사용:
```dart
class MyScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) { // ← 여기로 ref가 들어옴
    final listAsync = ref.watch(currentJusojungListProvider('A')); // 구독
    return listAsync.when(
      loading: () => Spinner(),
      error:   (e, _) => ErrorView(),
      data:    (list) => ListView(...),
    );
  }
}
// 버튼 콜백 — 기능 호출(구독 X)
ref.read(currentJusojungListProvider('A').notifier).togglePin(id);
```

### `extends _$...` 와 `@override build` 의 정체 (코드 생성)
- `@riverpod`를 붙이면 build_runner가 `*.g.dart`에 **추상 부모 클래스** `_$Xxx`를 생성한다. (`_`=private, `$`=생성물 관례)
- 그 부모 안 `build`는 **몸통 없는 abstract 함수**. "있어야 한다"고 선언만 함. → 자식(내 코드)이 `@override build`로 실제 내용을 채운다.
- 역할 분담: **부모(생성됨)** = 보관·갱신·캐싱·dispose 같은 복잡한 배관 전부 / **자식(나)** = "초기 데이터를 뭘로 채울지"의 핵심 로직만.
- `*.g.dart`는 직접 수정 금지. `.dart` 고치면 `flutter pub run build_runner build --delete-conflicting-outputs` 재실행. `part 'xxx.g.dart';`가 생성물을 내 파일의 일부로 붙여줘서 `_$Xxx`를 같은 파일처럼 쓸 수 있다.

### build는 생성자인가? → "재계산 함수"에 가깝다 (엑셀 수식)
- 생성자처럼 **처음 필요해질 때 자동 1회 실행**되는 건 맞다(내가 직접 호출 X).
- 하지만 생성자와 달리 **입력이 바뀌면 다시 실행(re-run)** 된다:
  1. 파라미터가 바뀌면(`build('A')`와 `build('B')`는 서로 다른 창고),
  2. 이 build가 watch하던 다른 Provider가 바뀌면,
  3. dispose 후 다시 필요해지면.
- 그래서 정확히는 "초기값을 계산하는 함수, 입력이 바뀌면 재계산" = 참조 칸이 바뀌면 자동 재계산되는 엑셀 셀 수식과 같다.

## Riverpod이 UI 여러 개를 어떻게 최적화하나 (깊은 버전)

- **① Provider는 값이 아니라 전역 식별자(키)다.** `xxxProvider` 변수는 "사물함 번호표"일 뿐, 실제 데이터/상태는 앱 전역 저장소(`ProviderContainer`)에 있다. `ref.watch(p)` = "그 사물함 열어 보여주고 + 내가 본다고 등록".
- **② 캐싱: 같은 Provider를 100개 위젯이 watch해도 데이터는 1개, `build()`도 1번만 실행.** 350ms 지연·서버 호출도 1번. 둘째부터는 계산 없이 캐시값 즉시 반환.
- **③ 선택적 리빌드: `state` 변경 시 그 창고를 구독한 위젯만 콕 집어 다시 그린다.** Riverpod이 `ref.watch` 호출들로 "위젯↔창고" 구독자 명단을 들고 있어서, 무관한 화면은 안 건드림.
- **④ `select`로 필드 단위 구독:** `ref.watch(p.select((x) => x.name))` → name이 실제로 바뀔 때만 리빌드(이전 값과 `==` 비교). 같은 창고라도 다른 필드 변경은 무시.
- **⑤ autoDispose: 구독자가 0명이 되면 창고 메모리 자동 회수.** 화면 떠나면 Provider 비워지고, 다시 들어오면 `build()` 재실행. (그래서 `ref.onDispose(...)`로 타이머 등 정리.)

## `ref` 가 뭐냐

- **`ref` = Provider 세계에 접근하는 리모컨**이자, **"누가 누구를 보는지" 의존성 그래프를 기록하는 펜**. ref 없이는 다른 창고를 열거나 구독하거나 청소 예약을 못 한다.
- 어디서 오나: (1) Provider/Notifier 안에선 내장돼 그냥 사용, (2) 위젯은 `ConsumerWidget`의 `build(context, ref)`로 주입받음.
- 주요 메서드:
  - `ref.watch(p)` — 구독(값 받고 바뀌면 자동 갱신).
  - `ref.read(p)` — 지금 한 번만 읽기(구독 X, 주로 콜백).
  - `ref.listen(p, cb)` — 바뀔 때 콜백 실행(화면 갱신 말고 팝업·네비 등).
  - `ref.onDispose(cb)` — 창고 치워질 때 청소 코드 등록.
- `ref.watch(B)`를 호출하는 순간 "지금 이 창고/위젯이 B를 의존한다"가 ref를 통해 기록되고, 이게 ③의 구독자 명단이 된다.

## 흔한 함정 / 헷갈리는 점

- **watch vs read 혼동:** 위젯 build 안에서 값 보여줄 땐 `watch`, 버튼 콜백에서 메서드 호출만 할 땐 `read(...).notifier`. build 안에서 read만 쓰면 갱신 안 됨, 콜백에서 watch 쓰면 안 됨.
- **`.notifier`:** 데이터가 아니라 **메서드(클래스)** 에 접근할 때 붙인다.
- **Provider 변수 ≠ 데이터.** 그건 키일 뿐. (최적화 ①)
- **`*.g.dart` 직접 수정 / build_runner 안 돌림** → `_$Xxx 없음` 류 에러. 거의 항상 코드 생성 누락.
- **파라미터 있는 Provider는 family**: `build(String scheduleId)`처럼 인자를 받으면 인자별로 별도 창고가 캐시된다.
- **async gap 주의:** `await` 뒤에서 위젯의 `ref`/`context` 접근 금지 → [[2026-06-05-riverpod-ref-context-async-gap]] 참고.

## 질문 기록
> 원칙 8.

### 내가 던진 질문 (원문)
- "riverpod가 뭐야"
- "음 내가 dart 랑 flutter 가 처음인데 rriverpod 부터 차근 차근 다시 설명해줄래? 이해하지 못했어 보다 자세히 설명해ㅑ줘"
- "1. 너가 보여준 예시 중 여기서 $로 extends 하는 이유와 build 를 overrides 하는 이유는 뭐야"
- "2. build 는 생성자처럼 이 클래스가 만들어질 ㅣ때 자동으로 채워지는건가"
- "3. riverpod 는 이렇게 ui 가 여러개를 물었을 때 어떻게 최적화를 진행했는지? 그 보다 상세한 깊이 까지 원해"
- "4. ref가 뭔지 모르겠다"

### Claude가 되물은 확인 질문 + 답
- (이번 세션에 Claude가 되물은 확인 질문 없음 — 개념 설명 위주 세션)

## 관련
- 발단이 된 이슈: (없음 — 학습 전용 세션)
- 관련 노트: [[2026-06-05-riverpod-ref-context-async-gap]], [[2026-06-06-flutter-dispose]], [[2026-06-06-flutter-진료실-데이터흐름-구조]]
- 코드 참조: lib/features/consultation/presentation/providers/current_jusojung_provider.dart (+ .g.dart)

---
- 생성일: 2026-06-06
- 마지막 갱신: 2026-06-06
