# 용어

- **건네주기(handoff)**: 한쪽(herdr pane / T3 thread)의 claude 대화를 다른 쪽에서 이어 가게 하는 것. 사용자가 원할 때 한 번만 한다.
- **fork**: 건네줄 때 받는 쪽이 `--fork-session`으로 갈라진 새 세션 ID를 쓰는 것. 원래 세션은 그대로 남는다.
- **옮기기**: fork가 안 될 때의 대안. 원래 쪽 claude를 멈추고 같은 세션 ID로 넘긴다.
- **세션 ID**: claude 세션 UUID. `~/.claude/projects/<cwd 인코딩>/<id>.jsonl`의 파일 이름.
  herdr에서는 `agent_session.value`, T3에서는 `nativeThreadRef.nativeId`.
- **thread**: T3 Code의 대화 단위. 가져온 thread의 ID는 `import:claudeAgent:<세션ID>`.
- **가져오기**: T3 내장 `importRecentAgentThreads`. 프로젝트 폴더의 최근 claude 세션을 thread로 만든다.
- **회사 맥 / 집 맥**: 같은 Tailscale tailnet에 있는 두 맥. 세션·herdr·T3 서버는 회사 맥에 있다.
