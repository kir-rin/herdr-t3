# 0001. herdr → T3 건네주기는 세션 사본 파일로 fork한다

2026-10-20 · 상태: 채택(사용자 결정)

## 배경

T3 Code의 가져오기(`importRecentAgentThreads`)는 claude 세션을 원래 세션 ID 그대로 thread로 만든다.
herdr pane의 claude가 그 ID로 계속 돌고 있을 때 T3에서도 같은 ID로 이어 가면, 두 claude가 같은
`~/.claude/projects/<cwd 인코딩>/<id>.jsonl`에 번갈아 써서 대화가 뒤섞인다.

T3 → herdr 방향은 `claude --resume <id> --fork-session`으로 새 ID를 받으면 된다. 하지만 herdr → T3
방향은 T3가 claude를 띄우므로 그 플래그를 넣을 수 없다.

## 검토한 방법

| 방법 | 파일 쓰기 | 단점 |
|---|---|---|
| **사본 파일 만들기** | `~/.claude/projects`에 새 jsonl 하나 | claude 내부 기록 형식에 기댄다 |
| T3 자체 fork (`canForkThread: true`) | 없음 | 가져온 뒤 첫 메시지 전에 사람이 T3에서 fork를 눌러야 한다 |
| 옮기기 | 없음 | herdr pane의 claude를 멈춰야 한다 |

## 결정

plugin이 원본 jsonl을 복사해 새 세션 ID의 사본을 만들고, T3가 그 사본을 가져오게 한다.

- 원본 파일은 읽기만 한다. 사본은 새 파일로만 쓰고, 기존 파일을 덮어쓰지 않는다.
- 사본 안의 세션 ID 필드를 새 ID로 바꾼다. 정확히 어떤 필드를 바꿔야 `claude --resume <새 ID>`가 되는지는
  구현 첫 단계에서 확인한다.
- 사본이 resume되지 않으면 이 결정을 다시 연다(대안: T3 자체 fork).

## 결과

- herdr worker는 멈추지 않고 계속 돈다. T3 쪽은 갈라진 새 세션으로 이어 간다.
- claude 버전이 올라 기록 형식이 바뀌면 사본 만들기가 깨질 수 있다. 사본을 만든 뒤 바로 resume을
  시험해서 실패하면 사용자에게 알리고 멈춘다.
- 가져오기는 원래 세션도 thread로 만든다. 원래 쪽 thread는 T3에서 이어 가지 않는다는 운용 규칙으로 둔다.
