---
title: "CLITrigger — AI CLI 에이전트를 위한 IDE를 만들었습니다"
categories: [AI]
toc: true
toc_sticky: true
---

AI 코딩 에이전트(Claude Code, Codex 같은 CLI 도구)로 개발하는 시간이 길어지면서 계속 걸리는 게 있었습니다. **도구가 전부 따로 논다**는 점입니다.

요구사항 문서는 노트 앱에, 작업 계획은 또 다른 툴에, 에이전트는 터미널 창 여러 개에, 결과물 확인은 git 클라이언트에. 에디터 중심의 개발에는 IDE라는 통합 환경이 있는데, CLI 에이전트 중심의 개발에는 그런 게 없었습니다. 매번 도구 사이를 오가며 맥락을 복사해 나르는 건 결국 사람이었고요.

그래서 직접 만들었습니다. [**CLITrigger**](https://github.com/HyperAITeam/CLITrigger) — AI CLI 에이전트를 위한 IDE, 말하자면 **AI 개발 커맨드 센터**입니다.

## 설계 철학: 하나의 파이프라인

CLITrigger의 뼈대는 단순합니다. AI에게 일을 시키는 과정을 다섯 단계의 **하나의 파이프라인**으로 잇는 것입니다.

```
문서(Docs) → 계획(Plan) → 터미널(Terminal) → 자동 작업(Tasks) → 형상관리(VC)
```

각 단계는 독립된 도구가 아니라 앞 단계의 맥락을 그대로 이어받습니다. 문서에 적은 요구사항이 계획이 되고, 계획이 에이전트의 프롬프트가 되고, 실행 결과가 리뷰 큐에 도착하는 것까지 — **의도(intent)가 단계 사이에서 유실되지 않게** 하는 것이 핵심 목표였습니다.

## 파이프라인 다섯 단계

### 1. 📚 Docs — 프로젝트 지식베이스

시작은 문서입니다. 프로젝트별 지식베이스에 Obsidian 스타일의 `[[wikilink]]`로 문서를 연결하고 그래프로 볼 수 있습니다. 중요한 건 이 문서들이 그냥 메모로 끝나지 않는다는 점인데, **원하는 문서만 골라 에이전트 프롬프트에 주입**할 수 있습니다. 요구사항·설계 문서가 곧 에이전트의 컨텍스트가 됩니다.

### 2. 🗓 Plan — 일정과 작업 계획

문서로 정리한 내용을 실행 가능한 단위로 쪼개는 단계입니다. 개인 일정 위에 프로젝트 일정과 작업 기한을 겹쳐 보는 **My Schedule**, 마크다운 임포트/익스포트가 되는 경량 **Planner**로 구성했습니다. 계획 도구를 무겁게 만들기보다, 파이프라인의 다음 단계로 자연스럽게 넘어가는 데 집중했습니다.

### 3. ⌨️ Terminal — 도킹되는 대화형 세션

계획을 다듬을 때는 에이전트와 직접 대화하는 게 빠릅니다. VS Code 스타일로 도킹되는 플로팅 창에서 장수명 CLI 세션을 유지할 수 있고, Claude / Antigravity / Codex를 **각자의 git worktree에서** 띄울 수 있습니다. 터미널 여러 개를 알트탭으로 오가는 대신, IDE의 패널처럼 배치해 놓고 쓰는 경험을 목표로 했습니다.

### 4. 🤖 Autonomous Tasks — 자동 실행

대화로 다듬은 작업을 이제 에이전트에게 위임하는 단계입니다. TODO마다 **격리된 git worktree**가 만들어져 서로 간섭 없이 병렬 실행되고, cron 기반 스케줄 실행과 rate limit 초과 시 자동 재시도를 지원합니다. 구현에 들어가기 전에 아키텍트·개발자·리뷰어 역할의 에이전트들이 먼저 토론하는 멀티 에이전트 모드도 넣었습니다. 실행할 CLI(Claude / Antigravity / Codex)와 샌드박스 모드도 작업 단위로 고를 수 있습니다.

### 5. 🔀 Version Control — 검토와 병합

파이프라인의 종착지입니다. 실행된 작업들의 diff를 **Review Queue**에서 한 화면으로 트리아지하고, 브라우저 기반 git 클라이언트에서 커밋·푸시·브랜치 관리까지 마무리합니다. 결과물을 확인하러 다른 도구로 나갈 필요가 없습니다.

## 파이프라인을 받치는 것들

- **Analytics** — 프로젝트별 비용·실행 통계
- **Live Logs** — WebSocket 실시간 로그 스트리밍
- **Remote Access** — Cloudflare Tunnel로 외부에서 접속
- **MCP Server** — CLITrigger 자체를 MCP 클라이언트에 노출
- **Favorites Launcher** — 자주 쓰는 외부 도구 원클릭 실행

스택은 Node.js + Express + TypeScript + SQLite 백엔드, React 18 + Vite + Tailwind 프론트엔드, 터미널은 node-pty + xterm.js 조합입니다. AI CLI 연동은 어댑터 패턴이라 특정 도구에 종속되지 않습니다.

## 설치

npm 한 줄이면 됩니다.

```bash
npm i -g clitrigger
clitrigger
```

`http://localhost:3000` 접속 → 비밀번호 설정 → 프로젝트 등록 → TODO 작성 → Start. 전제조건은 Node.js 22+, Git, 그리고 AI CLI 하나 이상입니다.

터미널 설치가 번거로우면 [릴리스 페이지](https://github.com/HyperAITeam/CLITrigger/releases/latest)에서 데스크톱 앱(Windows `.exe` / macOS `.dmg` / Linux `.AppImage`)을 받으면 됩니다. Node.js가 번들되어 있어 별도 설치가 필요 없습니다.

## 마치며

AI가 코드를 더 많이 쓰게 될수록 개발자의 일은 "의도를 정확히 전달하고 결과물을 검토하는 것"으로 옮겨간다고 생각합니다. CLITrigger는 그 흐름 전체 — 문서부터 병합까지 — 를 한 곳에서 다루기 위해 만든 도구입니다.

**MIT 라이선스** 오픈소스입니다. 써보시고 이슈나 피드백 남겨주시면 큰 힘이 됩니다. ⭐도 환영합니다.

👉 [github.com/HyperAITeam/CLITrigger](https://github.com/HyperAITeam/CLITrigger)
