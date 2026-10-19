# herdr ↔ T3 Code 대화 건네주기 — 설계

2026-10-20 · 상태: 초안(승인 전)

## 왜 만드나

herdr는 여러 agent를 돌리고 지켜보는 곳이고, T3 Code는 한 대화를 읽고 diff를 검토하는 곳이다(사용자 결정).
지금은 한쪽에서 하던 claude 대화를 다른 쪽에서 이어 갈 방법이 없다. 원할 때 한 번씩 대화를 건네주는
도구를 만든다.

쓰는 순간(사용자 결정):
- **herdr → T3**: worker가 끝났거나 blocked일 때, 그 대화를 T3에서 열어 결과를 읽고 diff를 검토한다.
- **T3 → herdr**: T3에서 대화로 방향을 정한 뒤, 오래 걸리는 실행을 herdr에 맡기고 다른 일로 넘어간다.

## 목표 / 하지 않을 것 / 실패로 볼 것

- 목표: 위 두 방향을 명령 한 번으로 한다. 회사 맥에서 돌고, 집 맥 T3 앱으로도 볼 수 있다.
- 하지 않을 것: 자동 동기화, 실시간 미러링, T3 화면 안의 버튼(T3에는 확장 기능이 없다, 아래 근거), codex 지원(v1).
- 실패로 볼 것: 원래 대화의 기록(`~/.claude/projects/**.jsonl`)이나 T3 DB가 깨지는 것.
  두 claude가 같은 세션 ID로 동시에 쓰는 것.

## 구성 (사용자 결정)

| 부품 | 위치 | 역할 |
|---|---|---|
| herdr plugin | 이 저장소 (`herdr-plugin.toml`) | **herdr → T3** action 하나 |
| claude 스킬 | 이 저장소 (`skills/…`), 전역 스킬로 배선 | **T3 → herdr**: T3 thread 안의 claude가 herdr에 직접 명령 |
| T3 서버 | 회사 맥, `t3 serve --tailscale-serve` | 집 맥 T3 앱이 "Add environment"로 붙는다 |

- claude 세션, herdr, T3 서버는 모두 **회사 맥 한 대**에 있다. 집 맥은 화면만 맡는다. 그래서 두 방향 모두
  같은 맥 안의 일이 된다. 회사 맥 서비스를 개인 tailnet에 여는 것은 사용자가 승인했다(2026-10-20).
- 건네줄 때는 **fork**한다(사용자 결정). 받는 쪽이 갈라진 새 세션으로 이어 가고, 원래 세션은 그대로 둔다.
  원래 쪽을 닫는 건 사용자 몫이다.
- 저장소는 `~/oss/herdr-t3`이고 개인 계정 `kir-rin`으로 커밋·push한다(사용자 결정).

## 조사로 확인한 사실 (2026-10-20, 집 맥 T3 Code Beta · 회사 맥 herdr 0.9.1 · Claude Code 2.1.303)

**herdr**
- `herdr agent list`(JSON)가 pane마다 `agent_session.value`(claude 세션 ID), `cwd`, `pane_id`, `agent_status`를 준다.
- plugin v1: `[[actions]]`(단축키 연결 가능), `[[events]]`, `[[panes]]`, `[[startup]]`, `[[link_handlers]]`를 manifest에 선언한다.
  명령은 argv 배열이고 shell을 거치지 않는다. herdr CLI 전체가 plugin API이며 `HERDR_BIN_PATH`로 부른다.
  문서: https://raw.githubusercontent.com/herdrdev/herdr/v0.9.5/docs/next/website/src/content/docs/plugins.mdx
- 새 pane을 열고 명령 실행: `herdr pane split`, `herdr pane run`.

**T3 Code**
- 데이터: `~/.t3/userdata/statev2.sqlite`. 이벤트를 쌓아 projection을 만드는 구조다(`orchestration_v2_events` + `*_projection_*`).
  **직접 쓰면 안 된다.** 읽기는 괜찮다.
- thread ↔ claude 세션: `orchestration_v2_projection_provider_threads.payload_json`의 `nativeThreadRef.nativeId`가
  claude 세션 ID다(`~/.claude/projects/<cwd 인코딩>/<id>.jsonl`과 일치 확인). cwd는
  `orchestration_v2_projection_provider_sessions.payload_json`의 `cwd`.
- 내장 가져오기: `importRecentAgentThreads`(WS 메서드 `agentSessionsImport`). 프로젝트 `workspaceRoot`의
  최근 claude 세션을 훑어 resume 가능한 thread `import:claudeAgent:<세션ID>`를 만든다. 같은 세션은 다시 가져와도 중복되지 않는다.
- CLI `~/.t3/bin/t3`: `serve`, `app <path>`(실행 중인 앱에서 프로젝트 열기), `project add|remove|rename`, `pair`.
  URL scheme `t3code://`(확인된 경로는 `t3code://app`뿐).
- 확장 기능 없음. 프로젝트 scripts는 `T3CODE_PROJECT_ROOT`, `T3CODE_WORKTREE_PATH`만 받고 thread ID는 받지 않는다.

## 동작

### herdr → T3 (plugin action)
1. 현재 pane의 claude 세션 ID와 cwd를 `herdr agent list`(또는 `pane current` + `agent get`)로 얻는다.
2. 원본 jsonl을 복사해 새 세션 ID의 사본을 만든다(사용자 결정, [ADR 0001](../../adr/0001-herdr-to-t3-fork-by-copy.md)).
   원본은 읽기만 하고 기존 파일은 덮어쓰지 않는다.
3. cwd가 T3 프로젝트가 아니면 `t3 project add <cwd>`로 추가한다. cwd가 git worktree면 그 worktree를
   별도 프로젝트로 추가한다(내 판단 — 가져오기가 프로젝트 `workspaceRoot` 기준이라).
4. T3 가져오기를 부른다(미확인 ②).
5. `t3 app <cwd>`로 **회사 맥 T3 데스크톱 앱**에서 그 프로젝트를 연다. 회사 맥은 데스크톱 앱과
   `t3 serve`를 함께 띄운다(사용자 결정). 집 맥 앱에서는 사용자가 직접 연다.
6. 결과(성공 / 무엇이 안 됐는지)를 herdr 알림으로 띄운다.

운용 규칙(내 판단): 가져오기는 원래 세션도 thread로 만든다. 막지 않고, 원래 쪽 thread는 T3에서 이어 가지 않는다.

### T3 → herdr (스킬)
"herdr로 넘겨" 같은 요청을 받으면 T3 안의 claude가 다음을 한다.
1. 자기 세션 ID와 cwd를 알아낸다. **추측하지 않는다**(내 판단): hook 등으로 확실한 ID를 얻지 못하면
   후보를 보여 주고 멈춘다. 같은 cwd에서 여러 claude가 돌 수 있어서 "가장 최근 jsonl"은 틀릴 수 있다.
2. 여는 위치(내 판단): 같은 cwd의 herdr workspace가 있으면 거기에 새 tab, 없으면 새 workspace.
   스킬은 herdr 밖에서 돌기 때문에 "현재 pane"이 없다.
3. 그 cwd에서 `claude --resume <id> --fork-session`을 원래 세션과 같은 권한 모드로 실행한다(내 판단 —
   권한이 다르면 무인 실행 중 바로 blocked). "넘기고 X 해"처럼 지시가 있으면 첫 프롬프트로 넣고
   (`herdr agent prompt`), 없으면 입력을 기다린다.
4. 새 pane ID를 답하고 턴을 끝낸다. 원래 T3 thread는 그대로 둔다.

## 미확인 — 구현 첫 단계에서 시험한다

1. **사본 파일의 resume.** jsonl 안의 어떤 세션 ID 필드를 바꿔야 `claude --resume <새 ID>`가 되는지.
   안 되면 ADR 0001을 다시 연다(대안: T3 자체 fork).
2. **가져오기를 스크립트에서 부르기.** WS 메서드라 인증(페어링 토큰)이 필요할 수 있다. 안 되면 앱의
   가져오기 버튼을 사용자가 한 번 누르는 걸로 대신한다.
3. **스킬이 자기 세션 ID를 아는 방법.** T3가 띄운 claude에 세션 ID가 노출되는지(환경변수·hook) 미확인이다.
   후보: cwd의 `~/.claude/projects/…`에서 가장 최근에 갱신된 jsonl.
4. 회사 맥에 `t3` CLI 설치, `t3 serve --tailscale-serve`와 집 맥 앱의 연결.

## 빌더에게 맡기는 것
plugin 구현 언어, 단축키 기본값, 알림 문구, 스킬 이름과 트리거 문구, 파일 배치.
