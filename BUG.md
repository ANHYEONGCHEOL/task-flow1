# BUG

근거: [SCENARIO-REVIEW.md](SCENARIO-REVIEW.md), [BUSINESS-RULE-REVIEW.md](BUSINESS-RULE-REVIEW.md), [DECISIONS.md](DECISIONS.md), [PRD.md](PRD.md), [ARCHITECTURE.md](ARCHITECTURE.md)

검수표의 "실제 결과"와 코드 읽기로 확인한 버그 목록이다. 2026-10-22 기준이다. 앱은 실행하지 않았고 테스트도 돌리지 못했다. (아래 "확인하지 못한 것" 참고)

## 요약
| ID | 제목 | 근거 | 원인 확인 |
|---|---|---|---|
| BUG-001 | 상태 변경이 저장 응답 전에 화면에 반영되고, 실패해도 되돌리지 않는다 | 코드 읽기 | 확인됨 (실행으로는 미확인) |
| BUG-002 | 등록한 카드가 새로고침 뒤 '진행 중' 열에 보인다 (S-04) | 검수표 실제 결과 | 미확인 |
| BUG-003 | '진행 중'을 골랐는데 카드가 '할 일' 열로 돌아온다 (S-05) | 검수표 실제 결과 | 미확인 |

## BUG-001 상태 변경이 저장 응답 전에 화면에 반영된다
- 증상: 카드의 상태를 바꾸면 API 응답을 기다리지 않고 카드가 먼저 새 열로 옮겨 보인다. 요청이 실패해도 목록을 되돌리지 않아, 저장되지 않았는데 카드가 새 열에 그대로 남는다. 새로고침하면 원래 열로 돌아간다.
- 위치: [frontend/src/App.tsx:19-27](frontend/src/App.tsx#L19-L27). `handleChangeStatus`가 `changeStatus` 호출 전에 `withTasks`로 목록을 바꾼다. 주석에 "연습용으로 D-004와 다르게 둠"이라고 적혀 있다.
- 재현 (실행해 보지는 않았다):
  1. 앱을 열고 카드를 하나 만든다.
  2. API 서버를 끈다.
  3. 카드의 상태 드롭다운에서 '진행 중'을 고른다.
- 기대 결과: 카드가 원래 열에 그대로 있고, 반영되지 않았음이 알려진다. (S-30)
- 예상 실제 결과: 카드가 '진행 중' 열에 보이고 오류 문구도 함께 나온다. 새로고침하면 '할 일' 열로 돌아간다. (S-35와 어긋남)
- 위반한 규칙: D-004 (저장이 끝난 뒤에만 성공), R-006 (화면 내용 = 저장 내용)
- 같이 영향받는 줄: S-30, S-33의 상태 변경, S-35 / BUSINESS-RULE-REVIEW의 D-004
- 수정 방향 (아직 고치지 않음): 수정·삭제처럼 `await changeStatus` 성공 뒤에만 목록에 반영한다.

## BUG-002 등록한 카드가 새로고침 뒤 '진행 중' 열에 보인다 (S-04)
- 증상: 카드를 등록하고 새로고침하면 '할 일'이 아니라 '진행 중' 열에 있다.
- 출처: SCENARIO-REVIEW.md S-04의 실제 결과 ("진행중 열로 이동", 판정 "차이").
- 기대 결과: 새로고침 뒤에도 '할 일' 열에 같은 카드가 보인다.
- 위반한 규칙: D-017 (새 카드의 처음 상태는 TODO), R-001, R-006
- 코드 확인: [backend/app/tasks.py:44-51](backend/app/tasks.py#L44-L51)은 등록 때 항상 `Status.TODO`로 저장한다. [frontend/src/components/Board.tsx:19](frontend/src/components/Board.tsx#L19)의 열 분류도 `COLUMNS`와 맞아 보인다. 코드만으로는 원인을 찾지 못했다.
- 원인 상태: **미확인.** 검수 때 실행한 서버·DB가 지금 코드와 같았는지, 검수 중 다른 조작이 끼었는지 알 수 없다.
- 확인 방법: 비어 있는 DB(`TASKFLOW_DB` 환경변수로 다른 파일 지정)로 서버를 켜고, 등록 직후 `GET /tasks` 응답의 status를 본다.

## BUG-003 '진행 중'을 골랐는데 카드가 '할 일' 열로 돌아온다 (S-05)
- 증상: '할 일' 카드의 상태 드롭다운에서 '진행 중'을 고르면 카드가 '할 일' 열로 돌아온다.
- 출처: SCENARIO-REVIEW.md S-05의 실제 결과 ("할 일 열로 돌아감", 판정 "차이").
- 기대 결과: 카드가 '진행 중' 열로 옮겨 보인다.
- 위반한 규칙: R-002, R-003, D-014, D-007
- 코드 확인: [backend/app/tasks.py:90-107](backend/app/tasks.py#L90-L107)는 받은 status를 검증해 저장하고, [App.tsx](frontend/src/App.tsx)와 [TaskCard.tsx:82-93](frontend/src/components/TaskCard.tsx#L82-L93)의 드롭다운도 `COLUMNS`의 값을 쓴다. 코드만으로는 원인을 찾지 못했다.
- 원인 상태: **미확인.** BUG-001의 먼저 반영하는 동작 때문에, 요청이 실패했는데 어떤 이유로 목록이 다시 불러와졌다면 이렇게 보일 수 있는지도 재현으로 확인해야 한다.
- 확인 방법: 브라우저 개발자 도구 Network 탭에서 `PATCH /api/tasks/{id}/status`의 요청 본문과 응답을 본다.
- 메모: BUG-002와 BUG-003이 같은 원인인지도 알 수 없다. 둘 다 카드의 상태와 열 배치가 어긋난다는 점만 같다.

## 확인하지 못한 것
- 검수표의 나머지 시나리오 37개(S-04, S-05 제외)와 BUSINESS-RULE-REVIEW.md의 D-001~D-020은 실제 결과가 비어 있다. "통과"가 아니라 "미확인"이다.
- BUG-002·003과 같은 원인이면 영향받을 수 있는 줄: S-06~S-10, S-12, S-16, D-007, D-014, D-017, D-020.
- 백엔드 테스트(backend/tests)를 돌리지 못했다. 이 PC의 `python`이 실제 파이썬이 아니고, 파이썬 환경은 설치하지 않기로 했다.
- 코드를 읽어서는 위 세 건 말고 D-004~D-020을 어기는 곳을 더 찾지 못했다. 읽은 파일: backend/app의 tasks.py · rules.py · schemas.py · db.py · main.py, frontend/src의 App.tsx · api.ts · types.ts · components 3개.
