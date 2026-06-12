# 2026-06-12 · 상병 설정 시트, 딤드·스와이프로 닫으면 저장·토스트가 안 됨

## 1. 개요
- 날짜: 2026-06-12
- 프로젝트 / 화면 / 모듈: devops-mediclo / 진료실 차트상세 · 상병 설정 바텀시트(`DiagnosisSheet`)
- 이슈 유형: 버그 (기획 스펙 미충족 / 변경분 유실)
- 계층: 클라이언트 (Flutter UI · 위젯 닫기 경로/라우트)
- 최종 상태: 해결
- 중요도: 보통

## 2. 환경
- OS / 기기 / 웹뷰: Flutter (태블릿 대상), 디바이스 무관(로직 결함)
- 앱 버전 / 빌드: develop 브랜치 작업 트리
- 브랜치 / 커밋: `develop` (directive 075 작업)
- 관련 의존성: Flutter / flutter_riverpod — 표준 위젯, 추가 의존성 없음

## 3. 증상
### 관찰된 현상
- 상병 설정 시트에서 상병을 추가/수정/삭제한 뒤
  - **X(닫기) 버튼**으로 닫으면 → 저장 + "저장되었습니다." 토스트 정상.
  - **딤드(어두운 배경) 영역 탭** 또는 **시트를 아래로 스와이프**해서 닫으면 → **저장도 안 되고 토스트도 안 뜸**. 변경분이 통째로 유실.
- 기획(Figma `489:13718` 공통 동작 정책)은 "저장 시나리오 : 입력란 엔터, **딤드 영역 선택, 바텀시트 아래로 스와이프**" 라고 세 경로 모두 저장 경로로 규정.

### 재현 절차
1. 진료실 차트상세 편집 모드 → 상병 박스 탭 → 상병 시트 열기.
2. 상병 1개 추가(또는 분류 변경).
3. X 버튼이 아니라 **딤드 영역을 탭**하거나 **시트를 아래로 스와이프**해서 닫는다.
4. 차트의 상병이 갱신되지 않고, 토스트도 안 뜬다. (변경 전 상태로 되돌아감)

### 에러 메시지 / 스택 트레이스 / 로그
```text
(런타임 에러 없음 — 예외 없이 "조용히" 저장이 누락되는 종류의 버그)
```

## 4. 확인한 내용
- 시트 호출부 `chart_detail_sections_body.dart` — `DiagnosisSheet` 를 `onChanged` **없이** 띄우고, `Navigator.push` 의 **반환값(pop result)** 으로만 저장하고 있었다.
  ```dart
  final result = await Navigator.of(context).push<List<Diagnosis>>(
    WriteSheetRoute(child: DiagnosisSheet(initialDiagnoses: ..., consultDate: ...)),
  );
  if (result != null) onDiagnosisChanged!(result); // ← result 가 null 이면 저장 스킵
  ```
- `WriteSheetRoute.onBarrierTap`(딤드 탭) 과 `BottomSheetContainer` 의 drag-dismiss(스와이프) 는 둘 다 `Navigator.pop()` 을 **인자 없이** 호출 → `result == null` → 위 `if` 가 false → 저장 스킵.
- 토스트(`저장되었습니다.`)는 시트 내부 `_close()`(=X버튼 핸들러)에 **만** 있었다. 딤드/스와이프는 라우트 레벨에서 바로 pop 하므로 `_close()` 를 안 거침 → 토스트도 누락.

### 결과 해석
원인이 **두 겹**으로 겹쳐 있었다 — (a) 저장값 전달을 "pop 반환값"에 의존, (b) 토스트가 X버튼 경로에만 묶임. X버튼만 두 조건을 동시에 만족해서 "X로는 되는데 나머지로는 안 되는" 모양이 됐다.

## 5. 원인 분석
- **확정 원인:** 시트 닫기 경로가 3개(X버튼 / 딤드 탭 / 아래 스와이프)인데, "저장+토스트" 로직이 **X버튼 경로(pop 반환값 + `_close`)에만** 연결돼 있었다. 라우트/컨테이너가 제공하는 딤드·스와이프 dismiss 는 결과 없이 pop 만 하므로 저장이 누락.
- **가능성 있는 원인:** 없음(정적 추적으로 단정 가능).

## 6. 조치 내역
기존 다른 작성 시트(주소증 `write_jusojung_sheet`, 기타 `write_etc_sheet`)가 쓰는 **`WriteSheetFlushable` + `WriteSheetRoute(beforeClose:)`** 패턴으로 세 경로를 단일 flush 로 수렴.

- `diagnosis_sheet.dart`
  - `_DiagnosisSheetState implements WriteSheetFlushable` 추가.
  - 저장+토스트 로직을 `flushPendingSave()` **한 곳으로 일원화**(SSOT). `_flushed` 가드로 닫기당 1회.
    ```dart
    bool _flushed = false;

    @override
    Future<void> flushPendingSave() async {
      if (_flushed) return;            // 이중 호출 가드 (닫기당 1회)
      _flushed = true;
      if (!mounted) return;
      if (_isChanged) {                // 변경 없으면 토스트/저장 모두 skip
        showToast(context, '저장되었습니다.');
        widget.onChanged?.call(_diagnoses);
      }
    }

    Future<void> _close() async {      // X버튼: flush 후 pop
      await flushPendingSave();
      if (mounted) Navigator.of(context).pop();
    }
    ```
- `chart_detail_sections_body.dart` — pop 반환값 의존을 제거하고, `GlobalKey` + `beforeClose` flush + `onChanged` 콜백으로 전환.
  ```dart
  final sheetKey = GlobalKey<State<DiagnosisSheet>>();
  Navigator.of(context).push<void>(
    WriteSheetRoute(
      beforeClose: () async {                          // 딤드/스와이프 dismiss 직전 호출됨
        final state = sheetKey.currentState;
        if (state is WriteSheetFlushable) {
          await (state as WriteSheetFlushable).flushPendingSave();
        }
      },
      child: DiagnosisSheet(
        key: sheetKey,
        initialDiagnoses: note.diagnoses,
        consultDate: _formatConsultDate(note.date),
        onChanged: onDiagnosisChanged,                 // 저장은 전적으로 콜백 경로로
      ),
    ),
  );
  ```
- **세 경로 수렴 확인:**
  | 경로 | 흐름 | flush |
  |---|---|---|
  | X버튼 | `_close()` → `flushPendingSave()` → `pop()` | ✅ |
  | 딤드 탭 | `WriteSheetRoute.onBarrierTap` → `beforeClose()` → `flushPendingSave()` | ✅ |
  | 스와이프 | `BottomSheetContainer` drag-dismiss → `beforeClose()` → `flushPendingSave()` | ✅ |

- 조치 결과: `flutter analyze lib/` 신규 경고 없음. 세 경로 모두 변경 시 저장+토스트, 무변경 시 토스트 미노출(닫기당 1회).

## 7. 최종 상태
- 해결 여부: 해결(정적 검증 기준). 실기기 수동 확인(세 경로 × 변경/무변경) 권장.
- 남은 문제: 같은 시트의 별개 후속(TC-8 드래그 정렬, TC-17/18 트리 아코디언, TC-26 성별검증)은 이 이슈 범위 밖.

## 8. 학습 포인트
- 핵심 개념: **"닫기 경로가 N개면 저장도 N개 경로 전부에서 보장돼야 한다."** `PopupRoute` 의 `barrierDismissible`(딤드)과 drag-dismiss(스와이프)는 개발자 코드를 안 거치고 pop 하므로, "닫히기 직전 훅(`beforeClose`)" 으로 **저장 지점을 한 곳에 모으는 게(flush 일원화)** 정석.
- 이번에 배운 점: 저장값을 `Navigator.pop(result)` 의 **반환값에 의존**하면, 개발자가 호출하지 않는 dismiss 경로에서 조용히 새어나간다. 콜백(`onChanged`) + `beforeClose` flush 조합이 더 견고.
- 다음에 같은 증상이면 확인할 순서: 1) 시트를 닫는 **모든** 방법을 나열(X/딤드/스와이프/뒤로가기) 2) 각 경로가 저장 로직을 거치는지 추적 3) 라우트의 `barrierDismissible`/drag-dismiss 가 `beforeClose` 훅을 호출하는지 확인.
- 👉 별도 학습 노트: [[2026-06-12-flutter-bottomsheet-close-paths-flush]]

## 9. 재발 방지
- 체크리스트:
  - [ ] 새 작성/편집 시트를 만들 때 닫기 경로(X·딤드·스와이프·시스템 뒤로)를 **전부** 적고, 각 경로가 flush 를 거치는지 표로 확인.
  - [ ] 저장값을 `pop(result)` 반환값에 의존하지 말고 `WriteSheetFlushable` + `onChanged` 로.
- 추가하면 좋을 테스트 / 가드: 위젯 테스트로 (a) 딤드 탭, (b) 드래그 dismiss 두 경로에서 `onChanged` 가 호출되는지 검증.
- 자동화 후보: `WriteSheetRoute` 를 쓰면서 `beforeClose` 가 비어있는 시트를 린트/grep 으로 탐지.

## 10. 트러블슈팅 가이드 (재발 시)
- 증상 신호: "X로 닫으면 저장되는데 배경 탭/스와이프로 닫으면 변경이 사라진다."
- 확인 절차: 시트 호출부에서 `WriteSheetRoute(beforeClose: …)` 가 채워져 있는지 → 시트 State 가 `WriteSheetFlushable` 을 구현하는지 → `flushPendingSave` 가 저장+토스트를 담는지.
- 조치 절차: 저장 로직을 `flushPendingSave` 로 옮기고, 호출부에서 `GlobalKey` + `beforeClose` 로 연결. X버튼도 `flushPendingSave` 경유로 통일.
- 정상 확인 기준: 세 경로 × (변경/무변경) = 6케이스에서 변경시 저장+토스트 1회, 무변경시 무토스트.

## 11. 질문 기록
### 내가 던진 질문 (원문)
- "딤드·스와이프 후속도 있어?"
- "로 내가 해당 코드를 이해하기 위해 예 코드를 만드는 것에 대한 프롬프트를 작성하도록 도와줄래? 그리고 현재 코드도 정리하고"

### Claude가 되물은 확인 질문 + 답
- Q: 이 후속(딤드/스와이프 저장 연동) directive를 발급할까요? / A: "ㅇㅇ ㄱㄱ" (발급+구현 진행)
- Q: QA 문서 TC-27도 바로 ✅로 갱신해둘까요? / A: ❓ 미답

## 12. 한 줄 결론
> 시트 닫는 길이 셋이면(X·딤드·스와이프) 저장도 셋 다 보장하라 — `pop` 반환값 대신 `beforeClose`+`WriteSheetFlushable` 로 저장을 한 점(flush)에 모은다.
