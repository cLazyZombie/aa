# 조사 근거와 세 도구 분석

조사일: 2026-10-16. 실행·설치 지시를 따라 하지 않고 고정한 소스를 읽었다. 원본은 git에서 제외한 `research/`에 있다. herdr-projects는 **v0.2.35만** 받아 분석했다. 다른 계열의 코드는 참고하지 않았다.

## 출처와 고정 버전

| 구분 | 출처 | 받은 버전·commit | 라이선스 |
|---|---|---|---|
| herdr | [공식 저장소](https://github.com/herdrdev/herdr/tree/v0.9.5) | v0.9.5, `2ecf19a92fc3a7fff75b6ce3655ef04f4625ec01` | Apache-2.0 |
| herdr-projects | [지정 tag](https://github.com/eliasstravik/herdr-projects/tree/v0.2.35) | v0.2.35, `1f05c002f22c8355733c567ef47c511612bcdb1f` | MIT |
| herdr-web-ui | [고정 commit](https://github.com/devswha/herdr-web-ui/tree/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741) | 0.4.10, remote runtime v29, `aa624cb56f550f9a7dd8bf4d59ed99ffc3921741` | MIT |
| 별도 보조 조사: mobile-relay | [설치본과 같은 tag](https://github.com/0cv/herdr-mobile-relay/tree/v0.21.3) | 0.21.3, `a58f1fc63469995534f4ae7de8e55220ce10ffaa` | **AGPL-3.0-or-later** |

공식 사이트의 저장소 연결로 herdr의 출처를 확인했다. mac-c 설치된 projects checkout도 `1f05c002…`였다. mac-a의 `~/.local/share/herdr-web-ui/remote-v29`는 `remote-v29-5c20f56c2190819f`를 가리켰다. 그 아래 server의 TypeScript 파일 100개를 복사하여 공개 저장소와 비교했고 **100개 모두 바이트 단위로 일치**했다. 따라서 설치 소스로만 추정한 분석이 아니다. mac-b mobile-relay의 release-manifest도 위 revision·darwin/arm64를 명시한다.

원격 운영 자료는 mac-b의 `~/.herdr-projects/projects/memory/{machines,preferences}.md`와 `library/t-0087/index.html`, `library/t-0088/index.html`을 SSH로 읽어 사본을 남겼다. 비밀 파일, 설정 전체, 인증서·키체인 항목은 가져오지 않았다.

## herdr 0.9.5

```mermaid
flowchart LR
  CLI[CLI / TUI clients] --> API[로컬 API socket · JSON lines]
  TUI[TUI / direct terminal] --> BIN[client socket · framed protocol]
  API --> SERVER[herdr server]
  BIN --> SERVER
  SERVER --> LAYOUT[workspace → tab → pane]
  SERVER --> PTY[PTY · shell · agent]
  PTY --> VT[ghostty VT · 화면·scrollback]
  VT --> BIN
  HOOK[agent integration] --> API
  SERVER --> SAVE[session JSON · history · snapshots]
  CLI --> SSH[SSH remote-client-bridge]
  SSH --> REMOTE[원격 herdr server]
```

- **기동·수명.** 단일 Rust 실행 파일에 CLI/client/server가 함께 있다. TUI 자동 감지 경로는 서버가 없을 때 별도 프로세스를 띄우며 `setsid`, stdio 분리를 적용한다. client detach는 pane을 종료하지 않는다. server 종료나 머신 재부팅은 실제 실행 중인 명령을 보존하지 못한다. 근거: [autodetect.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/server/autodetect.rs), [platform/mod.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/platform/mod.rs).
- **계층.** workspace는 프로젝트·checkout 단위, tab은 레이아웃 묶음, pane은 터미널 표현이다. terminal ID와 pane ID를 분리해 pane 이동과 터미널 수명을 구분한다. 다중 client와 화면 변경분 전송, 별도 terminal attach가 있다.
- **API·CLI.** `pane run/read/send-keys`, `workspace list`, agent 시작·상태·대기 등을 JSON request로 보낸다. brief의 `pane keys`에 대응하는 소스의 정식 명령은 `pane send-keys`다. 명령 표기는 해당 버전 CLI로 확인해야 한다. API는 로컬 socket, Unix 권한 0600이며 초기 request 제한은 1MiB다. 화면/client 프로토콜은 u32LE 길이 + bincode, protocol 22이고 일반 frame 상한은 2MiB다. endpoint generation 1은 내부 binary protocol과 따로 호환성을 관리한다. 근거: [API server](https://github.com/herdrdev/herdr/blob/v0.9.5/src/api/server.rs), [pane CLI](https://github.com/herdrdev/herdr/blob/v0.9.5/src/cli/pane.rs), [wire](https://github.com/herdrdev/herdr/blob/v0.9.5/src/protocol/wire.rs), [endpoint](https://github.com/herdrdev/herdr/blob/v0.9.5/src/protocol/endpoint.rs).
- **머신.** `machine add/list`로 저장한 머신을 `--machine`으로 선택한다. SSH stdio bridge가 원격 client socket으로 바이트를 전달한다. 중요한 결함은 `ensure_remote_server_running`이 서버 부재 시 **SSH 안에서 자동 기동**한다는 점이다. 웹이나 원격 클라이언트가 먼저 접근하면 잘못된 책임 프로세스·보안 세션으로 서버가 생길 수 있다. 근거: [remote/host.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/remote/host.rs).
- **상태 감지.** 프로세스 식별과 화면 패턴, 잠깐의 idle 전환을 억제하는 확인 절차를 결합한다. Claude·Codex의 공식 integration은 세션 식별·복원 정보를 보고한다. 화면 기반 working/blocked 판정과 hook 정보를 같은 신뢰도로 취급하면 안 된다. Codex hook v10은 SessionStart의 session 정보에 집중한다. 근거: [agent_detection.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/pane/agent_detection.rs), [Codex hook](https://github.com/herdrdev/herdr/blob/v0.9.5/src/integration/assets/codex/herdr-agent-state.sh), [Claude hook](https://github.com/herdrdev/herdr/blob/v0.9.5/src/integration/assets/claude/herdr-agent-state.sh).
- **복원.** `session.json`, `session-history.json`, `session-snapshots/`(48개 보관), `plugins.json`을 분리한다. 복원은 저장한 cwd에서 새 shell을 만들고 agent session ID 또는 보고된 resume argv로 재개한다. 원래 프로세스·빌드의 연속 실행을 뜻하지 않는다. 사용자 운영 기록에는 기본 resume가 model/effort/권한 옵션을 잃은 사례가 있다. 반면 소스에는 검증한 custom resume argv 경로도 있다. 새 설계는 실행 스펙과 resume 스펙을 명시적으로 함께 보존해야 한다. 근거: [restore](https://github.com/herdrdev/herdr/blob/v0.9.5/src/persist/restore.rs), [writer](https://github.com/herdrdev/herdr/blob/v0.9.5/src/persist/writer.rs), [agent_resume](https://github.com/herdrdev/herdr/blob/v0.9.5/src/agent_resume.rs).
- **플러그인.** TOML manifest의 build/startup/actions/events/panes를 외부 명령으로 실행한다. env에 plugin·workspace·pane·socket 문맥을 준다. 프로세스 제한·출력 제한은 있으나 임의 프로그램을 실행하는 신뢰 모델이다. 확장 개념은 가져오되 초기부터 플러그인 설치·자기 갱신까지 복제하지 않는다. 근거: [manifest](https://github.com/herdrdev/herdr/blob/v0.9.5/src/app/api/plugins/manifest.rs), [runtime](https://github.com/herdrdev/herdr/blob/v0.9.5/src/app/api/plugins/runtime.rs).
- **macOS 문맥.** `setsid`의 detached 여부와 Mach bootstrap 문맥, TCC 책임 프로세스, 키체인 상태는 별개다. herdr는 자동 기동 서버를 per-user bootstrap으로 옮기는 private API도 쓴다. `getsid()==pid`를 모든 플랫폼의 건강 판정으로 가져오면 안 된다. 근거: [bootstrap](https://github.com/herdrdev/herdr/blob/v0.9.5/src/platform/macos/bootstrap.rs).

가져올 것: 실제 PTY, workspace/tab/pane, terminal 수명 분리, CLI와 API의 같은 기능 범위, snapshot+event, session identity, bounded output. 버릴 것: 접근자가 서버를 임의로 자동 기동하는 경로, 내부 bincode enum에 공개 호환성을 의존하는 방식, 책임 계보를 운영자가 매번 복구해야 하는 방식.

### 이미지 붙여넣기 소스 확인

사용자 추가 지시에 따라 같은 **v0.9.5 commit**의 다음 경로를 확인했다. 실제 clipboard를 읽거나 운영 pane에 이미지를 입력하지 않았으며, 이 절은 코드 분석이다.

```mermaid
flowchart LR
  K[client · Ctrl+V / 빈 bracketed paste / 파일 drop] --> C[client OS clipboard 이미지 bytes]
  C --> W[ClipboardImage · target / extension / data]
  W --> SSH[기존 client transport · remote bridge]
  SSH --> S[대상 herdr server · 크기·target 검사]
  S --> F[대상 머신 temp 파일]
  F --> P[절대 경로를 pane / terminal / popup Paste로 삽입]
  P --> A[agent가 대상 머신 이미지 파일을 읽음]
```

| 확인 항목 | 소스에서 확인한 내용 |
|---|---|
| trigger·원격 구분 | [client/clipboard_images.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/client/clipboard_images.rs)의 `endpoint_accepts_local_images`는 원격 client/endpoint 및 활성 surface를 요구한다. `should_bridge_clipboard_image_paste`는 기본 설정 Ctrl+V와 빈 bracketed paste를 판별하며, Windows는 대응하는 input event를 처리한다. 로컬 endpoint의 모든 Ctrl+V를 이 bridge가 처리한다고 해석하면 안 된다 |
| 실제 입력 처리 | [client/mod.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/client/mod.rs)에서 이미지 수집 성공 시 `write_remote_image_to_server` 후 원래 key 전송을 건너뛴다. 이미지가 없으면 일반 입력으로 계속 진행한다. 원격 terminal에 붙일 로컬 image-file 경로 drop도 파일 bytes를 읽어 보낸다 |
| macOS 수집 | [platform/macos.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/platform/macos.rs)의 `read_clipboard_image`는 osascript로 PNGf clipboard를 `herdr-clipboard-image-<pid>-<nanos>.png`에 잠시 저장하고 bytes를 읽은 뒤 이 **client-side** 파일을 지운다. 아래 server staging 파일과 다른 단계다 |
| 다른 OS 수집 | [platform/windows.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/platform/windows.rs)는 OpenClipboard·등록 PNG format·CF_DIBV5/CF_DIB→PNG 경로다. [platform/linux.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/platform/linux.rs)는 wl-paste/xclip과 환경 DISPLAY/WAYLAND_DISPLAY를 사용한다. 이는 **client clipboard 수집**에 필요하며 원격 agent 머신의 clipboard를 공유하는 방식이 아니다 |
| wire·크기 | [protocol/wire.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/protocol/wire.rs)에 `ClientMessage::ClipboardImage`의 target/extension/data와 `MAX_CLIPBOARD_IMAGE_PAYLOAD=16MiB`가 있다. [server/client_transport.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/server/client_transport.rs)가 초과 payload를 거부한다 |
| 저장 경로·권한 | [server/clipboard_image.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/server/clipboard_image.rs)의 `stage`가 `<temp>/herdr-clipboard-images-<euid>/client-<client_id>-clipboard-<nanos>-<attempt>.<extension>`을 create_new로 만든다. PNG 수집이면 `.png`, 지원 확장자는 png/jpg/gif/webp/bmp다. Unix는 directory 0700/file 0600, Windows suffix는 UID가 아니라 **server PID**이며 이 파일의 restrict 함수는 no-op다 |
| agent 입력 | [server/headless.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/server/headless.rs)의 `ClientClipboardImage` 처리와 `paste_client_clipboard_image_path`가 target을 검증하고 **server-local 절대 경로**를 Pane Paste·Popup Paste 또는 DirectTerminal 입력으로 넣는다. [terminal_attach.rs](https://github.com/herdrdev/herdr/blob/v0.9.5/src/server/terminal_attach.rs)는 terminal mode에 따른 paste payload를 만든다. 이미지 전송을 sixel/terminal graphics 렌더링과 혼동하지 않는다 |
| 수명·한계 | 성공 staging 파일을 client의 staged_clipboard_files에 기록하고 client disconnect 때 지운다. 추가 stage 때 24시간 이상 된 파일도 정리한다. 따라서 client 수명과 agent의 지연 파일 읽기·resume 수명이 일치한다고 보장할 수 없다. 확장자 sanitize 자체는 수신 이미지 bytes의 완전한 decode 검증이 아니다 |

새 aa 계획에서는 수집은 client, 검증·파일 생성은 **대상 OS agentd**, 입력 형식은 harness adapter가 소유한다. local/remote를 같은 attachment API로 처리하고 SHA256·길이 receipt 후만 경로를 입력한다. 경로만 다른 머신으로 보내지 않고 bytes를 전송한다. client detach만으로 파일을 삭제하지 않으며 session/thread 참조·보관 정책을 둔다. Windows는 명시적 SID ACL, 세 OS native 경로·인용·이미지 소비 E2E를 적용한다. 이는 제품 구현이 아닌 [기획서의 필수 기능·아키텍처·검증 계획](plan.md)이다.

## herdr-projects v0.2.35

```mermaid
flowchart LR
  F[프로젝트 폴더 · AGENTS.md · memory · TASKS.md] --> C[coordinator agent]
  C --> CMD[projects CLI · profile · thread 명령]
  CMD --> H[herdr CLI / 일부 socket API]
  H --> W[thread · worktree · agent pane]
  W --> R[report.md · library]
  T[ticker] --> H
  T --> R
  T --> I[inbox · routine · sidebar]
  I --> C
  H --> M[herdr --machine · 원격 실행]
  T --> S[SSH · scp · rsync · 보고 회수]
```

프로젝트 폴더가 coordinator의 문맥이다. coordinator는 평범한 agent이며 결정은 문맥·skill을 따라 한다. 바이너리는 thread 생성, 보고 복사, 상태 분류, resolve 같은 결정적인 절차를 맡는다. 별도 프로젝트 데이터베이스는 없다. ticker는 별도 상주 루프다. 근거: [operations](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/docs/operations.md).

| 관심 항목 | 확인한 동작 | 새 프로그램에서의 선택 |
|---|---|---|
| profile | 이름·harness·model·effort·args, 허용 목록; 로컬과 원격 정의가 다를 수 있음 | 이름뿐 아니라 실제 해석 결과와 revision을 thread에 저장 |
| coordinator | 프로젝트 cwd로 발견; context를 매 turn 읽음 | 프로젝트 폴더·AGENTS.md·memory 중심 유지, event cursor 추가 |
| ticker | 기본 15초, brief 2초, 원격 약 1분; cheap pass 먼저, 느린 복사 뒤 | event subscription 우선 + 제한된 reconciliation timer |
| inbox | timestamp·kind·subject·counter 이름의 Markdown; 처리 완료 별도 폴더 | 잃지 않는 이벤트/ack, Markdown은 읽을 수 있는 투영으로 유지 |
| routine | every/daily·prompt·명령; 정확한 명령 내용 승인, DST 고려 | 동일 개념, 실행 ID·기한·missed-run 정책을 명시 |
| report | thread의 report hash가 달라지면 home에 복사; library 상한·symlink 검사 | hash 확인한 원자적 publish 후 review 이벤트; 파일 기록 유지 |
| 원격 thread | 모든 herdr 호출을 `--machine`으로 전달, profile은 SSH로 원격 resolve | 공통 typed agent API; 양쪽 profile 정책 검증 |
| placement | 원격은 repo가 있는 worktree만 허용; tab/checkout 원격 금지 | 원격 repo 없는 작업도 machine-local task dir에서 실행 |
| 상태 | starting/open/failed/resolved와 표시 group이 따로 있음 | 실행 상태·agent 관측 상태·검토 상태·연결 상태를 분리 |
| 정리 | report/library 최종 회수 후 worktree 제거; 불완전 복사는 보존 | receipt 검증 후만 정리, dirty worktree를 강제로 삭제하지 않음 |
| hook | SessionStart/UserPromptSubmit/PostToolUse로 context·progress 주입 | 작은 native hook CLI, 범위·세션 token 확인, 기존 설정 자동 변형 최소화 |

직접 확인한 핵심 소스: [herdr.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/herdr.rs)(socket binding·on_machine), [threads.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/threads.rs)(remote profile·placement·brief), [remote.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/remote.rs)(resolve_profile·SSH 인용·report hashes·copy), [ticker.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/ticker.rs), [thread.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/thread.rs), [inbox.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/inbox.rs), [routine.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/routine.rs), [setup.rs](https://github.com/eliasstravik/herdr-projects/blob/v0.2.35/src/setup.rs).

상태 파일 배치:

```text
~/.herdr-projects/<project>/
  PROJECT.md, AGENTS.md, MEMORY.md, memory/, TASKS.md, routines/
  threads/<id>.toml, <id>.md, <id>.task.md, <id>.next.md
  inbox/, inbox/done/, library/<id>/, .state/
<thread cwd>/.herdr-project/<project>-<id>/brief.md, report.md, library/
~/.herdr-projects/.ticker.lock, .ticker.log, .progress/
~/.config/herdr-projects/config.toml, owned.json, approved-routines.json
```

`Herdr::cmd`가 저장 socket을 env에 넣고 inherited HERDR_SESSION을 제거하는 점은 다중 세션 오접속 방지에 좋다. 반면 각 요청에서 CLI 프로세스를 만들고, 원격 profile·해시·보고 복사에 별도 SSH/scp/rsync가 필요하다. 동일 OS·POSIX 도구를 전제로 한 절차를 Windows에 그대로 옮길 수 없다. plugin manifest의 지원 플랫폼도 macos/linux다. 원격 셸 스크립트 대신 agent binary의 typed API로 경로·파일 회수·실행을 처리한다.

보존할 중요한 운영 철학은 **보고는 파일, 프롬프트는 알림**이다. 놓친 nudge 때문에 결과를 잃지 않아야 한다. 안전 설정은 같은 사용자 shell을 가진 agent에 대한 강한 보안 경계가 아니므로 새 프로그램도 profile 허용 목록을 sandbox라고 부르지 않는다.

## herdr-web-ui 0.4.10 / remote-v29

```mermaid
flowchart LR
  PHONE[모바일 PWA / 브라우저 · React · xterm.js] --> WEB[Bun HTTP / WebSocket server]
  WEB --> RPC[herdrRpc · JSON lines · socket/pipe]
  WEB --> NODE[Node PTY sidecar · terminal attach]
  RPC --> H[herdr server]
  NODE --> H
  WEB --> SSH[SSH · bundle 설치 · tunnel]
  SSH --> REMOTE[remote-entry.ts · Bun · loopback random port]
  REMOTE --> RH[원격 herdr socket]
  WEB --> RELAY[MachineRelay · 인증된 WebSocket 중계]
  RELAY --> REMOTE
```

- Bun이 API·WebSocket과 정적 frontend를 제공하고, React/Vite/xterm.js가 브라우저 화면을 만든다. transcript reader로 agent 대화를 chat 형태로 표시하고 실제 terminal로 전환한다.
- `server/herdr/client.ts`는 Bun.connect로 **JSON 한 줄/요청 한 연결**을 보낸다. Unix socket과 Windows named pipe/marker 차이를 처리한다. Unix의 terminal attach는 Node + `@lydell/node-pty` sidecar로 실행하며 흐름 제어를 제공한다. Windows에서는 이 버전의 herdr attach 지원 제약 때문에 화면 mirror를 사용한다. 세 OS 동일 기능이라고 추정하지 않았다.
- `remote-entry.ts`는 SSH로 기동되어 고정 bundle의 Bun에서 실행된다. 원격 loopback의 임의 port와 난수 token을 descriptor에 남긴다. server가 없으면 Unix는 Bun.spawn으로 herdr server를 띄우고, Windows는 SSH/Bun job에 묶이지 않도록 WMI로 띄운다. **macOS에서 이 자동 기동이 10-16 운영 사고와 직접 연결된다.**
- MachineRelay는 인증된 원격 WebSocket을 중계하고 xterm 처리 ACK를 통과시킨다. 출력량·정체 시간을 제한하고 overload는 명시 종료한다. 느린 모바일이 전체 PTY를 무한히 막지 않도록 bounded buffer와 재동기화가 필요하다.
- 인증은 shared token, HttpOnly·SameSite cookie, TLS에서 Secure, 장치별 pairing/권한, Tailscale identity를 조합한다. 빈 token이 곧 항상 안전함을 뜻하지 않으며 `access.ts`의 위치·identity 정책을 함께 봐야 한다. Origin 검사·변경 권한 재확인이 있다. 새 설계에서는 loopback에서도 웹 요청을 인증하고 proxy identity는 명시적으로 신뢰한 경로에서만 받는다.
- 모바일은 단순 CSS 축소를 넘는다. PhonePanel, 입력용 composer, IME composition 처리, 키보드 때문에 변하는 viewport, touch/pinch·고정 grid·비활성 tab의 resize 억제가 들어 있다. 초기 aa도 터미널만 작게 넣으면 모바일 사용성이 충족되지 않는다.
- **웹 이미지 paste.** Composer의 `onPaste`가 clipboardData file items를 모아 업로드하고 `imageMention`으로 `@path`를 초안에 넣는다. PaneTerminal의 terminal paste는 text-only 동작을 유지하며 file-only paste를 업로드한 뒤 경로를 붙인다. `useMachineApi`가 업로드 시점의 machine ID를 묶고 `/api/machines/<id>/pane/image`를 machine API가 원격 bridge의 `/api/pane/image`로 proxy한다. 실제 파일은 **대상 머신**의 `<pane cwd>/.herdr-web-ui/paste-<stamp>-<uuid>.<ext>`에 0600으로 쓰며 cwd 부재 시 temp fallback, HERDR_WEB_PASTE_DIR override가 있다. 웹 제한은 32MiB이고 base64 JSON body를 사용하므로 herdr native bridge의 16MiB와 같은 경로·제한이 아니다. aa는 streaming binary upload·공통 정책을 추천하며 agent별로 검증하기 전 `@path`나 POSIX single quote를 보편 규칙으로 가져오지 않는다.

근거: [index.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/index.ts), [herdr/client.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/herdr/client.ts), [pty/session.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/pty/session.ts), [remote-entry.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/remote-entry.ts), [machine-relay.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/machine-relay.ts), [auth.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/auth.ts), [access.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/access.ts), [TerminalInput.tsx](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/components/TerminalInput.tsx).

가져올 것: terminal/chat 전환 개념, 모바일 입력, 읽기/조작 권한, output ACK와 제한, 기기별 연결 상태. 줄일 것: Bun→Node→herdr의 다층 실행, 웹 서버의 agent 서버 자동 설치·기동, 거대한 vendor별 transcript 파서를 MVP에 모두 포함하는 범위. Rust 웹 gateway가 기존 agent API에 붙고 PTY 생성은 하지 않도록 한다.

이미지 흐름 추가 근거: [Composer.tsx](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/components/Composer.tsx), [PaneTerminal.tsx](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/components/PaneTerminal.tsx), [compose.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/lib/compose.ts), [machineContext.tsx](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/lib/machineContext.tsx), [api.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/src/lib/api.ts), [machine-api.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/machine-api.ts), [paste.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/server/paste.ts), [attachments.ts](https://github.com/devswha/herdr-web-ui/blob/aa624cb56f550f9a7dd8bf4d59ed99ffc3921741/shared/attachments.ts).

## mac-b mobile-relay는 별도 제품이다

설치본의 README와 manifest를 확인한 뒤 같은 v0.21.3 소스를 별도로 받았다. Go server, Svelte frontend, per-device controller/reader, 암호화 WebSocket·gateway·WebRTC 경로를 쓴다. herdr-web-ui의 `MachineRelay`와 이름이 비슷하지만 코드·라이선스·배포가 다르다. Native Windows는 이 버전의 지원 대상이 아니다.

외부 gateway까지 다루는 E2EE/NAT 우회 기능은 현재 사용자 네트워크의 기본 요구에 넣지 않는다. AGPL 코드·frontend·protocol 구현을 MIT 프로젝트에 섞지 않고, 장치 취소·권한·연결 상태 같은 일반 개념만 참고한다. 향후 그 코드를 실제로 가져오려면 별도 라이선스 결정과 해당 의무 검토가 먼저 필요하다. 근거: [LICENSE](https://github.com/0cv/herdr-mobile-relay/blob/v0.21.3/LICENSE), [transports](https://github.com/0cv/herdr-mobile-relay/blob/v0.21.3/docs/transports.md), [deviceauth](https://github.com/0cv/herdr-mobile-relay/blob/v0.21.3/internal/deviceauth/store.go).

## 기존 스택 비교와 재사용 범위

| 계층 | herdr 0.9.5 | projects 0.2.35 | web-ui 0.4.10 | aa 추천 |
|---|---|---|---|---|
| 언어·실행 | Rust, Tokio+threads | Rust, 동기 명령·ticker | Bun TS + Node sidecar | Rust 공통 core/agent/gateway, TS frontend |
| 터미널 | portable-pty vendor, ghostty-vt, Ratatui/Crossterm | herdr CLI 호출 | xterm.js, node-pty, Windows mirror | portable-pty+플랫폼 adapter, ghostty-vt adapter, Ratatui, xterm.js |
| 연결 | local socket/pipe, JSON API, binary client, SSH | herdr --machine + ssh/scp/rsync | HTTP/WS, SSH bridge, relay | versioned typed API, UDS/pipe, SSH stdio + outbound WSS |
| 상태 | JSON snapshot/history | Markdown/TOML/JSON+lock | 파일 기반 기기·환경설정, transcript store 읽기 | SQLite 상태·event outbox + Markdown 문맥·보고 |
| 웹 | 자체 서버 없음 | popup/sidebar | React 19, Vite, Bun.serve | React/Vite/xterm.js + Rust Axum |
| 배포 | 플랫폼별 단일 binary, Zig VT 빌드 | plugin binary, macos/linux | Bun/Node/bundle 조합 | OS별 agent binary, Mac AAHost.app, 서명 manifest |

herdr의 VT binding은 Apache-2.0이고 내부 libghostty-vt는 MIT다. vendor metadata의 upstream commit은 `44f2a44df7e8c4a0c6df3f7d872ef3d7ead88e51`이며 이 소스는 Zig 0.16.0을 요구한다. build script에 macOS/Linux/Windows target 매핑이 있지만 **우리 환경에서 세 OS를 빌드한 것은 아니다**. native CI로 검증하기 전까지 호환 가능성과 실제 합격을 구분한다.

Apache-2.0/MIT 소스의 선택적 재사용은 각 라이선스·저작권·필요한 NOTICE 및 수정 표시를 보존하는 조건으로 계획한다. 현재 저장소의 기존 MIT LICENSE를 유지했다. 아직 upstream 코드를 제품에 복사하지 않았다. 세 도구 전체를 fork해서 결합하기보다 공통 domain/API를 만들고 검증한 소형 부분만 가져오는 편이 업데이트·중복 상태 문제를 줄인다.

## 실제 운영에서 가져온 요구

mac-b가 coordinator·ticker를 맡고 mac-a/mac-c은 agent를 실행한다. 10-16 기준 운영 조합은 herdr 0.9.5 + projects 0.2.35다. 오래된 memory의 0.2.34 줄보다 같은 파일의 11:55 업데이트 기록과 설치 checkout을 우선했다. 무조건 update 명령을 실행하지 않았다.

10-16 mac-a 재부팅 뒤 remote-v29 Bun이 먼저 herdr를 띄워 잘못된 계보·detached=false·키체인 locked가 생겼다. 그때 개발자 도구 probe는 OK였다. 따라서 **빠른 첫 실행 하나만으로 서비스가 건강하다고 판정할 수 없다.** t-0087의 초기 LaunchAgent 추천은 t-0088의 detached/경합 조사와 이후 실제 운영 교훈으로 수정됐다. 오래된 권고를 새 프로그램의 검증된 해결책으로 재사용하지 않는다.

사용자는 file 보고·검토 대기, profile별 실행, thread 격리, 중단한 빌드의 성공 오인 방지, 운영 프로세스 보호를 중시한다. 새 프로그램의 migration은 현재 thread를 강제 재개하거나 종료하지 않고, 읽기 전용 inventory → 새 작업만 aa → 기존 thread 자연 종료 순서로 잡는다.
