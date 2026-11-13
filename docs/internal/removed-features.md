# Herdr 사내 빌드: 제거·차단 기능 목록

Herdr 사내 빌드는 `herdr-app`(데스크톱 앱)과 `herdr-core`(`hc`, 터미널 런타임)를 포크해, 로컬 기능은 유지하고 외부로 나가는 경로를 막은 빌드다.

- 브랜치: 두 저장소 모두 `internal`
- 방식: **코드는 남기고 끈다.** 업스트림 동기화를 쉽게 하려고 대부분 상수 스위치로 막았다.
  - 앱: `src/update.rs`의 `pub const ONLINE: bool = false`
  - 코어: `crates/herdr-core/src/machines.rs`의 `pub const REMOTE_ENABLED: bool = false`
  - 상수라서 설정이나 환경변수로는 다시 켤 수 없고, 바꾸려면 재빌드해야 한다.
  - 변경 지점에는 모두 `INTERNAL BUILD` 주석이 있다(`git grep "INTERNAL BUILD"`).
- 검증 상태는 문서 끝의 [검증 현황](#검증-현황)을 따른다.

## 한눈에 보기

| 영역 | 처리 |
|---|---|
| 로컬 터미널, 탭·분할, 레이아웃, 에이전트 상태 표시, 로컬 git, 파일 패널 | 유지 |
| mod (JS 확장) | 유지. 단, 네트워크 접근은 차단하고 **검토한 사내 mod만 설치**(운영 정책) |
| 원격 머신 (SSH, `via`, 다른 코어의 접속) | 차단(코어). 앱 화면의 진입점 제거 |
| 텔레메트리 | 제거(전송 코드 삭제, 수집 항상 꺼짐) |
| 피드백 전송 | 제거(전송 코드 삭제, 명령·버튼 비활성) |
| 앱·코어 자동/수동 업데이트, 코어 설치 다운로드 | 차단. 설정에는 버전만 표시 |
| 릴리스 노트 미디어 다운로드 | 차단(내장 `CHANGELOG.md` 텍스트만 표시) |
| 중국 계열 에이전트 (DeepSeek Harness, Kimi, Qoder) | 제거 |
| 터미널 OSC 52 클립보드 쓰기 | 기본 꺼짐(설정으로 켤 수 있음) |
| 공개 릴리스 CI (R2 업로드, 서명·공증) | 워크플로 삭제 |
| 외부 mod 공유 안내 (herdr.dev, 공개 GitHub) | 사내 저장소 안내로 교체 |

## 1. 원격 머신

| 기능 | 처리 | 위치 |
|---|---|---|
| ssh·`via` 명령 실행, 대상 검사, 로그인·로그아웃 | 거부 (`machines::disabled()`) | core `machines.rs` |
| 원격 Windows 설치(scp) | 거부 | core `machines.rs` (`install_windows`) |
| 부팅 시 저장된 머신에 재접속 루프 | 다이얼하지 않음. 저장된 머신은 unreachable로만 표시 | core `remote.rs` (`keep_up`) |
| 다른 코어의 접속(hello의 `core`) | 거부 | core `server.rs` |
| 설정 > Machines 섹션 | 제거 | app `defaults/frame/settings/main.js` |
| 웰컴의 "다른 머신" 페이지(4쪽 → 3쪽), 소개 문구 | 제거 | app `defaults/frame/welcome/main.js` |
| 사이드바 하단 "Add machine" | 제거(하단 높이 69 → 41px) | app `defaults/mods/workspace-chrome/main.js`, `defaults/layouts/main.json` |
| `machine-add` mod (`~/.ssh/config` 읽기, 머신 추가 대화상자) | 삭제. 기존 설치본은 앱 시작 시 자동으로 지워짐(수정하지 않은 경우) | app `defaults/mods/machine-add/`, 관련 렌더 테스트, `scripts/machine-add.test.ts` |
| 에이전트 안내 문서 | "원격 머신은 꺼져 있음"을 명시 | app `defaults/AGENTS.md`, core `src/SKILL.md`, `skills/herdr-core/SKILL.md` |

- `machines.json`에 예전에 저장된 머신은 디스크에 남지만 연결하지 않는다.
- 로컬 연결은 유지된다: Unix 소켓, Windows named pipe(원격 클라이언트 거부, SDDL로 SYSTEM과 현재 사용자만 허용), `hc connect`.

## 2. 텔레메트리·피드백·업데이트

| 기능 | 처리 | 위치 |
|---|---|---|
| 사용 데이터 전송 (`telemetry.herdr.dev`) | `post` 삭제, `endpoint()`는 None, 수집 항상 꺼짐. 쌓인 큐는 삭제 | app `src/telemetry.rs` |
| 설정 > Privacy(텔레메트리 토글) | 제거 | settings `main.js` |
| 피드백 전송 (`telemetry.herdr.dev/v1/feedback`) | `post` 삭제, 항상 실패 | app `src/feedback.rs` |
| `app.feedback` 명령, `herdr-app feedback` CLI, 설정의 Send feedback 버튼, 웰컴 문구 | 명령은 즉시 실패, 버튼·문구 제거 | `window.rs`, `control.rs`, `main.rs`, `commands.rs`, settings·welcome |
| 앱 업데이트 확인·다운로드 (`releases.herdr.dev`) | `checks_app()`는 None, `curl_text`·`unpack` 스텁 | app `src/update.rs` |
| 코어 설치·번들 다운로드 | `ONLINE` 검사로 실패 | app `src/core_bin.rs` |
| 앱의 `hc update` 호출(`app.update_core`, 앱 업데이트 직후) | `update_to`·`after_update`에서 차단 | app `core_bin.rs`, `update.rs` |
| `hc update`, 원격 설치용 다운로드 | `releases::curl` 스텁 | core `releases.rs` |
| 릴리스 노트 미디어 (`media.herdr.dev`) | 다운로드 안 함 | app `src/update_notes.rs` |
| 설정 > Updates | "About"으로 바꾸고 버전과 What's new만 표시 | settings `main.js` |
| 설치 화면의 `irm …\| iex` / `curl \| sh` 안내 | 사내 배포 안내 문구로 교체 | app `src/setup.rs` |

## 3. mod

| 항목 | 처리 |
|---|---|
| `fetch`, WebSocket, TCP, TextView 원격 이미지 | 차단 (`NetworkGrant::Denied`, `src/mods.rs`) |
| `image("https://…")`, Avatar URL | 간접 차단. gpui 기본 `NullHttpClient`를 쓰고 앱이 `with_http_client`를 부르지 않기 때문. **HTTP 클라이언트(`reqwest_client` 등)를 추가하면 안 됨** |
| 프로그램 실행 (`exec`, `ExecuteGrant::Unrestricted`) | **유지.** 기본 mod(files, changes 등)가 sh·git·hc를 실행해야 함. 그래서 mod는 `curl` 등으로 네트워크 차단을 우회할 수 있으므로, **검토한 사내 mod만 설치하는 운영 정책(A안)**으로 통제한다 |
| AGENTS.md의 herdr.dev·공개 GitHub에서 mod 받기·공유하기 안내 | 사내 mod 저장소에서만 받고, 사내 저장소에 제출하도록 교체 |

## 4. 에이전트 연동

- 제거(중국 계열): DeepSeek Harness(`dsh-tui`), Kimi(`kimi`), Qoder(`qoder`).
  - `defaults/programs.json` 항목, `defaults/icons/` 아이콘, `win98/taskbar.js` 항목
- 유지: claude, codex, copilot, gemini, cursor, opencode, pi, pig, amp, grok, antigravity, kiro, hermes, crush, 이름 매칭만 하는 항목(mastracode, muse, command-code, prime-agent, ollama).
- 이 목록은 이름 매칭과 실행 버튼만 담당한다. hook을 설치하지 않고, 통신도 하지 않는다. 코어에는 에이전트 목록 자체가 없다.
- 사용자가 직접 설치한 CLI 실행이나 OpenCode 같은 도구의 모델 선택은 앱이 막을 수 없다. 네트워크 정책으로 통제한다.

## 5. 보안 기본값

| 항목 | 처리 |
|---|---|
| 터미널 OSC 52 클립보드 쓰기 | 기본 `false`. `settings.json`의 `"clipboard_write": true`로 켤 수 있음. 클립보드 읽기는 원래부터 불가 |
| 코어의 OSC 52/8/9 통과 | 코어는 그대로 전달만 함. 실제 처리는 앱의 위 설정이 결정 |
| 터미널 링크 Ctrl/Cmd-클릭 | 유지(사용자 클릭 시에만, 모든 스킴). 필요하면 http(s)와 file로 제한 |

## 6. CI·배포

| 항목 | 처리 |
|---|---|
| `.github/workflows/release.yml` (두 저장소: R2 업로드, Azure 서명, Apple 공증) | 삭제. core의 `production_signing_covers_both_release_targets` 테스트는 `#[ignore]` |
| 이슈 템플릿의 Discord·herdr.dev·GitHub 보안 링크 | 제거 |
| PR CI (`ci.yml`) | 유지 |

## 남은 작업 (결정 필요)

| 항목 | 이유 |
|---|---|
| `distribution/install.sh`, `install.ps1` (두 저장소, `releases.herdr.dev`에서 다운로드) | 사내 배포 주소가 정해지면 바꾸거나 삭제. 지금은 사용자가 직접 실행할 때만 동작 |
| app `ci.yml`의 `curl https://bun.sh/install` | 사내 미러로 교체 |
| app `scripts/notes-media.ts` (R2 업로드 개발 도구) | 자격 증명이 있어야 동작함. 삭제 검토 |
| core `scripts/fetch-conpty.ps1` (GitHub에서 nupkg), ghostty 서브모듈과 Zig 의존성 | 빌드할 때 받는 것들. 사내 미러에 두고 오프라인 빌드를 확인해야 함 |
| 배포 패키징 워크플로 | 아직 작성 안 함 |
| 컴파일 단계에서 제거 (`#[cfg(feature = "remote")]`) | "바이너리에 원격 코드가 없어야 한다"는 요구가 생기면 상수 대신 사용 |

## 이 포크가 통제하지 않는 것

이 빌드는 기능 제거 포크이지, 통신 샌드박스가 아니다. 다음이 만드는 통신은 네트워크 정책과 CLI·모델 설정으로 통제한다.
- 터미널에서 실행하는 AI CLI와 셸 명령
- MCP 서버
- mod가 `exec`로 실행하는 프로그램

## 검증 현황

| 항목 | 상태 |
|---|---|
| 정적 감사 (두 저장소: URL, 프로세스 실행, 소켓, 의존성) | 완료. 코어에는 HTTP·TLS 라이브러리와 TCP·UDP 리스너가 없음 |
| 수정한 JS 문법 검사 (`node --check`: settings, welcome, workspace-chrome) | 통과 |
| `programs.json` JSON 검사 | 통과 |
| `cargo build` / `cargo test` | **미실행** (작업 PC에 Rust 툴체인 없음). Rust 수정은 검토만 함 |
| 앱 실행 확인, 네트워크 관찰(DNS·HTTP) | 미실행 |
