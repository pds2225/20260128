<!-- BEGIN OPS HEADER: 실행 게이트. 본문보다 우선. -->
# TASK.md — 이 레포 실행 단일 기준

```text
REPO: pds2225/20260128
BASE: main
```

## 0. STOP 게이트
- 시작 전 `git fetch --all --prune` 및 origin URL/현재 브랜치/dirty 상태를 확인한다.
- 실행 지시는 이 `TASK.md`만 사용한다. 다른 레포 TASK/NEXT_TASK/옛 채팅 과업을 가져오지 않는다.
- 기본 브랜치에서 직접 개발하지 않는다. destructive git, force push, secret/.env 수정 금지.
- 이 TASK가 명시하지 않으면 기본 브랜치에 병합하지 않는다.
<!-- END OPS HEADER -->

# TASK — 20260128

## 목표
현재 등록된 과업 없음. 새 과업이 이 파일에 명시되기 전에는 구현하지 않는다.

## 현재상태
- 기본 브랜치: `main`
- CURRENT TASK: 없음

## 구현범위
- 없음

## 금지사항
- 임의 기능 개발·리팩터링·다른 레포 작업 수행 금지.
- 테스트 삭제/skip으로 성공 처리 금지.

## 입력검증
- 저장소가 `pds2225/20260128`인지 확인한다.
- 원격 `main` 최신화 후 CURRENT TASK 존재 여부를 확인한다.

## 빈상태
- CURRENT TASK가 없으면 `NO_ACTIVE_TASK`만 보고하고 즉시 종료한다.

## 로딩상태
- 향후 UI/비동기 기능 과업에서는 기존 구조에 맞는 loading 상태를 반드시 구현·검증한다.

## 오류상태
- 오류를 숨기지 않는다. 재현 증거와 함께 `BLOCKED` 또는 `FAIL`로 보고한다.

## 테스트
- 과업이 생기면 targeted test 후 필요한 전체 테스트를 실행한다.

## 회귀검증
- 기존 정상 기능 회귀 여부를 확인한다.

## 문서동기화
- README/TASKS/관련 문서가 실제 구현과 불일치할 때 필요한 범위만 갱신한다.

## Git 규칙
- 독립 작업은 별도 branch/worktree로 병렬 가능.
- 별도 지시가 없으면 구현 → 테스트 → commit → push까지만 수행하고 `main` 병합 금지.

## DONE/BLOCKED
- DONE: 구현·필수 테스트·회귀검증·필요 문서동기화가 모두 통과한 경우만.
- BLOCKED: 요구 불명확, 필수 의존성 없음, 테스트/환경 실패, 사용자 결정 필요 시.

## 최종보고
```text
REPO:
BRANCH:
COMMIT:
PUSH:
CHANGED:
TEST:
REGRESSION:
STATUS: DONE | BLOCKED | FAIL | NO_ACTIVE_TASK
```

## 실행지시
원격 `main`을 최신화하고 이 `TASK.md`를 처음부터 끝까지 읽은 뒤, 여기에 적힌 과업만 수행한다.
