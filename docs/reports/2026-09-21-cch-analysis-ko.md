# Claude Code Harness 전수조사 & 수익화 분석 (한국어)

> 작성일: 2026-09-21
> 대상 저장소: <https://github.com/bmshin94/claude-code-harness>
> 원본(업스트림): <https://github.com/Chachamaru127/claude-code-harness>
> 분석 버전: v5.15.0
> 분석 방식: 저장소 전체(1,752 파일) 실측 조사

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [이게 뭐 하는 물건인가](#2-이게-뭐-하는-물건인가)
3. [폴더별 전수조사 결과](#3-폴더별-전수조사-결과)
4. [동작 흐름 (쉬운 설명)](#4-동작-흐름-쉬운-설명)
5. [언제 쓰는가](#5-언제-쓰는가)
6. [장점과 단점](#6-장점과-단점)
7. [Q&A — 설치/정체/토큰/유명세/로컬에이전트/React·PHP](#7-qa)
8. [수익화 아이디어 8종](#8-수익화-아이디어-8종)
9. [추천 로드맵](#9-추천-로드맵)

---

## 1. 기본 정보

| 항목 | 내용 |
|---|---|
| 프로젝트명 | Claude Code Harness (CCH) |
| 원본 저장소 | `Chachamaru127/claude-code-harness` (일본 개발자, 사실상 1인 개발) |
| 이 저장소 | `bmshin94/claude-code-harness` — 업스트림 포크 (커밋 200개 보존) |
| 버전 | **v5.15.0** (최근 릴리즈 2026-09-06, 개발 매우 활발) |
| 라이선스 | **MIT** (상업적 이용/수정/재배포/사유화 모두 허용) |
| 규모 | 파일 1,752개 / Go 99,546줄 + Shell 91,773줄 + 문서(md) 36,414줄 |
| 주 언어 | 코드·README는 영어, 내부 문서/주석 상당수가 일본어 |
| 포크 고유 커밋 | `a54d93e docs: appended CLAUDE.md persona guide` (페르소나 추가) |

---

## 2. 이게 뭐 하는 물건인가

> **한 줄 정의**: AI 코딩 도구(Claude Code / Codex CLI / Cursor / Grok)가 제멋대로 굴지 못하도록
> **"계획 → 작업 → 검수 → 출시"** 레일을 깔아주는 개발 플러그인.

"harness(하네스)"는 원래 말에 씌우는 마구(고삐)라는 뜻이다. 이름 그대로 **AI라는 말에 고삐를 채우는 장치**.

### 설계 철학 3가지

1. **계획을 사람이 쓰는 게 아니라, 사람이 승인한다** — AI가 `spec.md` + `Plans.md`를 만들고 사람은 승인/수정만.
2. **구현자와 검수자를 물리적으로 분리한다** — Worker가 "다 했다"고 해도 기억을 공유하지 않는 별도 Reviewer가 재검증.
3. **강제력은 마크다운이 아니라 컴파일된 코드에 둔다** — 마크다운 규칙은 설득·망각이 가능하지만 Go 바이너리의 `DENY`는 협상 불가.

---

## 3. 폴더별 전수조사 결과

### 3.1 `skills/` — 스킬 23개

핵심은 **5개 동사(verb)** 뿐이다.

| 스킬 | 하는 일 | 게이트 |
|---|---|---|
| `/harness-plan` | 의도 → `spec.md` + `Plans.md` (범위·완료조건·의존성·불확실성) | 사람이 승인/수정 |
| `/harness-work` | 승인된 계획 실행 (`3` = 3번만, `all` = 전체) | TDD 필수 조건 검사 |
| `/harness-review` | **구현과 분리된** 독립 리뷰 | Major 발견 시 완료 차단 |
| `/harness-sync` | 계획 vs 실제 구현의 **drift(괴리)** 검출 | 증거 기반 상태 보정 |
| `/harness-release` | CHANGELOG + 태그 + GitHub Release | 릴리즈 프리플라이트 통과 필수 |

나머지 18개: `breezing`(팀 병렬 실행), `harness-loop`(자동 반복), `memory`(SSOT 기억),
`harness-accept` / `harness-progress` / `harness-plan-brief`(비개발자용 화면 3종),
`cursor-ask` / `cursor-do` / `cursor-review` / `cursor-setup`, `ci`, `failure-codifier`(실패 패턴 학습),
`agent-browser`(브라우저 검증), `session-send`(세션 간 메시지), `cc-update-review`, `maintenance`,
`harness-setup`, `japanese-writing-drafter`.

### 3.2 `agents/` — 서브 에이전트 5종

| 에이전트 | 모델 / effort | 역할 |
|---|---|---|
| `worker.md` | Sonnet 5 / medium | 실제 구현. **worktree 격리**, `Agent` 툴 금지(재귀 방지), maxTurns 100 |
| `reviewer.md` | Sonnet 5 / **xhigh** | 독립 리뷰 전담, 쓰기 권한 없음 |
| `advisor.md` | Fable 5.1 / high | 상담역 (동일 실패 2회 반복 시 자동 호출) |
| `test-wiring-auditor.md` | — | 테스트 조작 감시관 (read-only, 고정 프롬프트) |
| `livemsg-gate.md` | — | 세션 간 메시지 검증 관문 |

### 3.3 `go/` — Go 네이티브 엔진 (약 10만 줄, 진짜 심장부)

**가드레일 R01–R16 (소스에서 실측 추출)**

| 규칙 | 차단 대상 |
|---|---|
| R01 | `sudo` 실행 |
| R02 / R03 | 보호 경로 쓰기 (settings 자기 수정 방지) |
| R04 | 프로젝트 밖 쓰기 → 확인 |
| R05 | `rm -rf` / `find -delete` → 확인 |
| **R06** | `git push --force` |
| R07 / R08 | Codex 모드·Breezing Reviewer의 쓰기 금지 |
| R09 | 비밀 파일 읽기 경고 |
| R10 | git 우회 플래그(`--no-verify` 등) |
| **R11** | 보호 브랜치에서 `git reset --hard` |
| **R12** | `main`/`master` 직접 push → 확인 |
| R13 | 리뷰 보호 경로 경고 |
| **R14** | src 수정 시 테스트 필수 |
| **R15** | 비밀 파일 `git add` 차단 |
| R16 | 자기 자신 승인 금지 |

**런타임 플로어 5종 (전역 OFF 스위치 없음)**

`money-billing`(결제) / `egress`(외부 전송) / `secret-read`(비밀 읽기) /
`prod-deploy`(운영 배포) / `worktree-escape`(작업공간 탈출)

그 외 Go 패키지 40여 개: `policy`, `guardrail`, `hookhandler`, `hookcodec`, `session`,
`breezing`, `judgmentledger`, `blastradius`, `scopeleash`, `auditlog`, `eventstore`,
`deliveryidentity`, `runtimefloor`, `retiredalias`, `autoapprove` 등.

### 3.4 `bin/` — 사전 빌드 바이너리 4종

`harness-darwin-arm64`, `harness-darwin-amd64`, `harness-linux-amd64`, `harness-windows-amd64.exe` (각 약 13MB).
`bin/harness` POSIX 셸 샴이 OS/아키텍처를 감지해 dispatch. **Node.js 불필요.**
바이너리가 없으면 stderr에 진단만 내고 **exit 0 + 빈 stdout** → 훅이 "판단 없음"으로 처리해 안전하게 통과.

### 3.5 `hooks/` + `.claude-plugin/hooks.json` — 자동 개입

`PreToolUse` 매처 `Write|Edit|MultiEdit|Bash|Read` 에 걸려 **실행 직전** Go 엔진이 판정.
추가로 `Write|Edit` 에는 **Haiku 모델 agent 훅**이 붙어 하드코딩 비밀·TODO 스텁·인젝션 취약점을 실시간 스캔.
그 외 `AskUserQuestion` 정규화, 파일 리스(lease) 훅, 브라우저 MCP 가이드 훅.

### 3.6 `tests/` — 테스트 217개

`validate-plugin.sh`(구조 검증), 3-CLI 훅 플로어 테스트,
가드레일 **우회 시도** 테스트(env 접두사 / sudo 래핑 / watch 조합 9케이스) 등 자기 보안을 자기가 검증.

### 3.7 멀티 호스트 지원

| 도구 | 등급 | 설치 |
|---|---|---|
| Claude Code / Codex CLI / Cursor / Grok | ✅ 정식 지원 | 플러그인 마켓 / `scripts/setup-*.sh` |
| OpenCode | 🟡 호환 이용 가능 | `scripts/setup-opencode.sh` |
| Codex app / Hermes Agent / GitHub Copilot CLI | 🔬 시험 대응 | 수동 |
| Antigravity CLI | ❌ 미지원 | — |

`hosts.toml` 하나가 각 호스트의 네이티브 훅을 **생성(`harness gen`)** 해 동일 정책 엔진으로 라우팅한다(복제가 아님).

### 3.8 `.claude/rules/` — 자기 규율 문서 20여 개

`test-quality.md`(테스트 조작 절대 금지), `defense-layer-blast-radius.md`(방어층 추가 전 5점 체크 — 실제 사고 2건 반성문),
`commit-safety.md`, `self-audit.md`(deny 규칙 감소 감지), `shared-file-discipline.md`(병렬 worktree 공유파일 규약) 등.

### 3.9 기타

- `templates/` — 스키마(`worker-report.v1`, `review-result.v1`, `livemsg-gate.v1`), 보안 baseline, 로케일, HTML 템플릿
- `docs/` — 100+ 문서 (아키텍처, 모델 라우팅, 호환성 매트릭스, 업스트림 추적 스냅샷 등)
- `scripts/` — 164개 셸/JS 스크립트 (CI, 코덱스 컴패니언, 미러 동기화, 증거 수집 등)
- `.github/workflows/` — 7개 (validate-plugin, codeql, scorecard, release, benchmark, smoke-install, opencode-compat)

---

## 4. 동작 흐름 (쉬운 설명)

### 비유: "AI 개발자를 고용한 건설 현장"

사용자는 건물주, Claude는 일 잘하지만 가끔 사고치는 신입. CCH는 **현장소장 + 안전관리자 + 감리** 세트.

### 장면 1 — 계획

```
/harness-plan 중복 주문 버그 고쳐줘. 완료 조건은 같은 주문이 한 번만 저장되는 것.
```

기존 코드/설계를 **먼저 읽고** `spec.md` + `Plans.md` 생성:

```markdown
### Phase 1. 중복 주문 수정
- [ ] cc:todo  1.1 중복 발생 재현 테스트 작성
- [ ] cc:todo  1.2 주문 저장 로직에 멱등성 키 추가
- [ ] cc:todo  1.3 동시 요청 부하 테스트

완료조건(DoD): 동일 주문 100회 동시 요청 → DB에 1건
unknown: 기존 주문 테이블 인덱스 구조 미확인
```

여기서 **정지**. 사람이 승인/수정하기 전에는 구현하지 않는다.

상태 마커: `pm:requested → cc:todo → cc:wip → cc:done → pm:approved` (`cc:withdrawn`은 종료 상태)

### 장면 2 — 실행

```
     ┌─ Worker A (worktree-1) ─ 1.1 테스트 작성 ─┐
Lead ┤                                           ├→ 통합(cherry-pick)
     └─ Worker B (worktree-2) ─ 1.2 로직 수정 ──┘
```

각 Worker는 독립 worktree를 받아 파일 충돌이 원천 차단되고, `worker-report.v1` 스키마로 **자기 점검 5건 필수** 보고.

### 장면 3 — 독립 검수 (핵심)

```
Worker: "완료했습니다"
   ↓
Reviewer (별도 AI, 기억 비공유, 쓰기 권한 없음)
   ↓
"1.2에 멱등성 키는 있으나 동시성 테스트 부재 → REQUEST_CHANGES"
   ↓
Worker 재작업 → 재검수 (반복 한도 내)
```

AI에게 자기 코드를 리뷰시키면 대체로 통과시킨다. 그래서 **기억을 공유하지 않는 별도 심판**을 세우는 것이 CCH의 핵심 아이디어.

### 장면 4 — 상시 가드레일

```
rm -rf ./build              → 프로젝트 내부, 통과(기록 남김)
git push --force origin main → R06 DENY
git add .env                → R15 DENY
운영 서버 배포               → prod-deploy 플로어 (끌 수 없음)
```

### 3층 구조

```
3층: 스킬 23개        (사람이 부르는 명령)      — 마크다운
2층: 에이전트 5종      (AI 역할 분담)           — 마크다운
1층: Go 엔진 10만 줄   (R01–R16 + 플로어 5종)   — 컴파일된 바이너리, 협상 불가
```

### 비개발자용 화면 3종

| 화면 | 시점 | 내용 |
|---|---|---|
| Plan Brief | 계획 확정 | AI가 이해한 내용 / 선택지 / 리스크 / 완료조건 |
| Progress | 작업 중 | WIP·TODO·완료 개수 + 결정 대기 항목 |
| Acceptance | 출시 전 | 조건별 통과·실패 + ship / wait / reject |

주의: 진행률은 **작업 개수 비율**이며 품질 통과율이 아니다(문서에 명시됨).

---

## 5. 언제 쓰는가

| 상황 | CCH의 대응 |
|---|---|
| 자는 동안 자동으로 돌리고 싶다 | `/harness-loop` 최대 8사이클, 막히면 advisor 상담 |
| AI가 파일을 날릴까 불안하다 | R05/R06/R11 + `worktree-escape` 플로어 |
| AI가 "다 했다"고 거짓 보고한다 | 독립 Reviewer + `/harness-sync` drift 검출 + 증거 없으면 `unknown` 표기 |
| 작업 20건을 하나씩 시키기 번거롭다 | `/breezing` 팀 병렬 실행 (worktree 분리) |
| 코드 못 읽는 사람에게 보고해야 한다 | HTML 3화면 자동 생성 |
| Claude ↔ Codex ↔ Cursor 를 오간다 | 동일 정책이 4개 도구에 적용 |

---

## 6. 장점과 단점

### 장점

1. **AI 삽질 방지 시스템을 통째로 획득** — 직접 만들면 수개월. MIT라 그대로 사용 가능.
2. **AI 워크플로 설계의 교과서** — 200커밋 + 문서 3.6만 줄에 "왜 그렇게 했는지(Why)"가 기록됨.
3. **Go 엔진이 실무급** — 정책 엔진, 훅 프로토콜, 세션 스토어 등 에이전트 인프라 레퍼런스.
4. **자기참조적 도그푸딩** — 하네스로 하네스를 개발. Phase 144까지 진행된 실전 증거.

### 단점

| 단점 | 설명 |
|---|---|
| 일본어 장벽 | 내부 문서·주석 상당수가 일본어. README만 영어 |
| 과도한 무게 | 스킬 23 + 규칙 20 + 문서 100+ → 1인 개발자에겐 학습곡선이 가파름 |
| 1인 프로젝트 | 버스팩터 1 |
| 미래형 모델 ID | `claude-fable-5-1`, `gpt-6-astra`, `gpt-5.6-luna` 하드코딩 → 실환경에서 라우팅 조정 필요 가능 |
| 포크 유지비 | 업스트림이 계속 갱신되므로 주기적 sync 필요 |

---

## 7. Q&A

### Q1. 설치 및 사용법

**Claude Code (30초)**

```bash
claude
/plugin marketplace add Chachamaru127/claude-code-harness   # 또는 bmshin94/...
/plugin install claude-code-harness@claude-code-harness-marketplace
/harness-setup
```

**첫 사용**

```bash
/harness-plan README 온보딩 흐름 개선          # spec.md + Plans.md 생성 → 승인
/harness-work all                              # 구현 + 자동 리뷰
/harness-sync                                  # 계획 vs 실제 차이 확인
/harness-release                               # 릴리즈(선택)
```

**다른 도구**

| 도구 | 설치 |
|---|---|
| Codex CLI | `scripts/setup-codex.sh --user` → Codex 재시작. 호출은 `$harness-plan` |
| Cursor | `scripts/setup-cursor.sh` |
| Grok | `scripts/setup-grok.sh` |
| OpenCode | `scripts/setup-opencode.sh` (런타임 동등성 미보장) |

**요구사항**: Claude Code v2.1+ / 쓰기 권한 있는 git 저장소 / **Node.js 불필요** / (선택) harness-mem

**자주 쓰는 명령**

```bash
/harness-work 3                 # 3번 작업만
/harness-work all               # 전체
/harness-work --codex           # Codex 백엔드
/breezing                       # 팀 병렬 실행
/harness-loop  |  status | stop # 자동 반복 / 확인 / 중단
/harness-progress               # 진행 HTML
/harness-work --resume latest   # 중단 지점 재개
bin/harness doctor --migration-report   # 진단(삭제 없음)
bin/harness session list        # 활성 세션 목록
```

### Q2. 플러그인인가, 스킬인가, MCP인가

**정답: Claude Code "플러그인"이며, 그 안에 스킬·에이전트·훅·바이너리가 모두 들어 있다.**

```
Plugin (.claude-plugin/plugin.json)
├─ Skills × 23      (skills/*/SKILL.md)
├─ Agents × 5       (agents/*.md)
├─ Hooks            (hooks/hooks.json)
├─ Go 바이너리 × 4   (bin/harness-*)
├─ Output Styles    (output-styles/)
└─ Scripts × 164    (scripts/)
```

**MCP는 아니다.** 다만 MCP를 **소비**하고(harness-mem 기억, Codex MCP 2차 리뷰, Playwright 브라우저)
**통제**한다(`mcp__codex__*` deny, 외부 MCP 서브에이전트 격리 정책).

> 정리: MCP = 도구 연결 규격 / 스킬 = 사용설명서 / 플러그인 = 둘을 담은 상자. CCH는 "상자".

### Q3. API 토큰이 필요한가

**CCH 자체는 불필요.** Go 바이너리가 로컬 실행되고 외부 호출이 없다.

| 항목 | 토큰 |
|---|---|
| CCH 플러그인 | 불필요 |
| Claude Code / Codex / Cursor / Grok | 각 도구의 기존 인증 그대로 |
| GitHub Release 기능 | `gh` CLI 인증 (릴리즈 시에만) |
| harness-mem | 불필요(로컬) |
| 웹 검색 / 스크래핑 | 사용 시에만 (샌드박스 allowlist 문서 제공) |

오히려 **토큰을 보호**하는 쪽이다: R09(비밀 읽기 경고), R15(`.env` staging 차단), `secret-read` / `egress` 플로어.
문서에는 "설정과 비밀이 섞인 디렉터리를 통째로 막으면 도구가 죽는다"는 실제 사고 사례
(`~/.config/gh`, `~/.npmrc`, `~/.ssh`, `~/.docker/config.json`, `~/.aws`, `~/.kube`)가 경고로 남아 있다.

### Q4. 왜 GitHub에서 주목받는가

> 참고: 이 분석 세션에서는 스타 수를 직접 확인하지 못했다. 아래는 코드·문서 기반의 근거.

1. **"AI 신뢰 문제"를 정면으로 푼 드문 프로젝트** — 다른 프로젝트가 프롬프트(마크다운)를 팔 때, CCH는 컴파일된 엔진으로 강제한다.
2. **실제 동작하는 멀티툴 지원** — Claude + Codex + Cursor + Grok을 단일 정책으로.
3. **과대광고를 하지 않는다** — "4개의 설치 경로 ≠ 4개의 동일한 보장", "`not_observed != absent`", "PR-ready는 release-ready가 아님" 같은 문장이 README에 명시. 지원 등급을 4단계로 솔직하게 구분.
4. **자기참조적 도그푸딩** — 하네스로 하네스를 개발한 Phase 144까지의 이력 자체가 증거.
5. **압도적 문서량** — `decisions.md`(왜), `patterns.md`(어떻게), `governance-rationale.md`(근거). 실패까지 기록.
6. **타이밍** — AI 에이전트 붐 + "AI가 사고칠까 두렵다"는 공포에 정확히 대응.

### Q5. 로컬 에이전트 구축에 도움이 되는가

**매우 도움된다.** 로컬 에이전트 레퍼런스로는 최상급.

| 재사용 가능 요소 | 위치 | 가치 |
|---|---|---|
| 훅 프로토콜 구현 | `go/internal/hookhandler`, `hookcodec` | stdin-JSON 규격, exit code 의미, 4개 호스트 차이 흡수 |
| 정책 엔진 | `go/internal/policy`, `guardrail` | 셸 명령 파싱 + 우회 방어(env 접두사, sudo 래핑, 백슬래시 이스케이프, `git -C`) |
| 에이전트 격리 | worktree isolation | 병렬 AI 충돌 방지 |
| 세션 스토어 | `go/internal/session`, `deliveryidentity` | `git --git-common-dir` 공유, `os.Link` create-only 락, TTL+active.json 이중 생존판정 |
| 스키마 계약 | `templates/schemas/*.json` | AI 출력을 파싱 가능하게 강제 |
| 감사 로그 | `judgmentledger`, `auditlog` | 차단 사유 사후 추적 |

**코드보다 값진 설계 원칙**

1. 강제력이 강한 층일수록 적용 범위를 좁게 — `permissions`(에이전트만) < hook(에이전트만) < `sandbox`(OS 전체)
2. `excludedCommands`는 서브프로세스에 **상속되지 않는다** — 제한은 상속되고 면제는 상속되지 않는 비대칭
3. user scope로 올리기 전에 1개 프로젝트에서 검증 — 전역 설정 = 전역 사고
4. 신뢰 경계를 명시 — `session_id`는 암호학적 보증이 없으므로 어디까지 믿는지 표로 정리

**추천 학습 순서**: `policy`+`guardrail` → `hooks.json`+`hookhandler` → `worker.md`+스키마 → 자기 도메인용 규칙 3~5개 재설계.
**주의**: 23개 스킬을 전부 켜지 말 것. 가드레일 층만 분리해 쓰는 것이 가장 실용적.

### Q6. React나 PHP로 만들 수 있는가

| 층 | React(Node) | PHP | 평가 |
|---|---|---|---|
| 스킬/에이전트(마크다운) | 가능 | 가능 | **언어 무관** — 텍스트 파일 |
| 훅 핸들러 | 가능 | 가능 | stdin JSON → stdout JSON |
| 정책 엔진 | 가능하나 느림 | 가능하나 느림 | Go 채택 이유 = 시작 속도 |
| 대시보드 UI | **React가 우위** | 가능 | 현재는 정적 HTML |

**왜 Go였나**: 훅은 도구 호출마다 매번 실행된다(하루 수천 회).
Go ≈ 5ms / Node ≈ 80ms / PHP ≈ 50ms / Python ≈ 120ms 수준의 프로세스 시작 비용 차이가 누적된다.
또한 단일 바이너리 배포 = 사용자가 런타임을 설치할 필요가 없다.

**권장 하이브리드 구조**

```
React + Next.js   (대시보드 / 승인 UI)     ← 직접 개발
Node 또는 PHP API (세션 상태 서빙)          ← 직접 개발
CCH Go 바이너리   (훅 / 정책 판정)          ← 그대로 재사용
```

> 결론: 다시 만들지 말고 **"Go 엔진은 빌려 쓰고, 그 위에 사람이 보는 층을 React로 얹는다."**

---

## 8. 수익화 아이디어 8종

### 전제: 라이선스

MIT — 상업적 이용 / 수정 / 재배포 / 사유화 모두 허용. 조건은 저작권 표시 + MIT 전문 유지.
즉 **SaaS로 만들어 과금해도 법적으로 문제없다**(원작자 크레딧 명시 권장).

### 핵심 인사이트: 무엇을 팔 것인가

CCH 자체는 무료다. 팔아야 할 것은 **CCH가 못 하는 것**이다.

| CCH가 잘하는 것 | CCH가 못하는 것 = 기회 |
|---|---|
| 로컬에서 규칙 강제 | 팀 전체 통제(중앙 정책) |
| 텍스트 로그 | 시각화 / 리포트 |
| 단독 동작 | 다인 협업 |
| 일본어 문서 | 한국어/영어 온보딩 |
| CLI | 모바일 승인 |
| 판단 기록 | 감사·규제 대응 증빙 |

---

### 아이디어 1. AI 코딩 거버넌스 SaaS ★★★★★ (1순위)

**컨셉**: "AI가 우리 회사 코드베이스에서 무엇을 했는지 CTO가 한 화면에서 보는 대시보드"

**해결하는 고통**: 개발자 10명이 각자 AI를 쓰는데, 누가 무엇을 시켰고 무엇이 차단됐으며
프로덕션 DB에 손댄 적이 없는지 경영진이 알 방법이 없다.

**아키텍처**

```
개발자 PC                         SaaS
Claude Code + CCH Go 엔진  ──텔레메트리(익명화)──→  React 대시보드
                                                   ├ 실시간 세션 뷰
                                                   ├ 차단 이벤트 피드
                                                   ├ 팀 정책 배포
                                                   ├ 감사 로그 검색
        ←──────── 정책 푸시 ────────                └ 주간 리포트 PDF
```

**가격**

| 플랜 | 가격 | 대상 |
|---|---|---|
| Free | $0 | 개인 1명, 로컬 대시보드 |
| Team | $29/사용자/월 | 5~50명, 중앙 정책, 감사로그 30일 |
| Business | $79/사용자/월 | SSO, 감사로그 1년, SOC2 리포트, 커스텀 규칙 |
| Enterprise | 별도 협의(연 $50k+) | 온프레미스, 전용 지원 |

**수익 시뮬레이션** (50명 팀 × $29 × 12개월 = 고객사당 연 약 $17,400)

- 보수적: 고객사 10곳 → 연 약 1.7억 원
- 현실적: 고객사 30곳 → 연 약 5.2억 원
- 낙관적: 고객사 100곳 → 연 약 17억 원

**MVP 3개월**: (1개월) `judgmentledger`/`auditlog` → JSON 송신 훅 + Next.js 대시보드 →
(2개월) 팀 정책 중앙 배포(`harness.toml` 원격 fetch) → (3개월) 주간 PDF + Slack 알림 + Stripe 결제

기술 난이도 ★★★ / 시장 ★★★★★ / 경쟁 ★★ → **종합 1순위**

---

### 아이디어 2. "AI 작업 증빙" 컴플라이언스 서비스 ★★★★★

**컨셉**: "이 코드는 AI가 작성했고 사람이 검수했다는 증거를 발급한다"

**배경**: EU AI Act(사용 이력 기록 의무), 금융·의료 감사(AI 코드 검증 절차 요구),
공공 SI 입찰(AI 사용 정책 제출), 보험(사고 시 과실 입증).

CCH는 이미 `judgmentledger`(판단 기록), `auditlog`(감사 로그), `review-result.v1`(리뷰 증거 스키마),
Acceptance 화면(조건별 통과/실패)을 갖고 있다. 이를 **증명서 형태로 포장**하는 사업.

**가격**: 프로젝트당 증명서 ₩500,000 / 월 구독(무제한) ₩2,000,000 / 감사 대응 컨설팅 ₩10,000,000 건당

단가 높음, 경쟁 거의 없음, 단 B2B 엔터프라이즈 영업이 난관 → **아이디어 1의 상위 플랜으로 묶는 것을 권장**

---

### 아이디어 3. 한국어 특화 포크 + 교육 ★★★★ (진입장벽 최저)

**컨셉**: "CCH-KR: 한국 개발자를 위한 AI 개발 하네스"

문서 3.6만 줄 중 상당수가 일본어라는 점이 곧 기회다.

| 상품 | 가격 | 내용 |
|---|---|---|
| 오픈소스 포크 | 무료 | 한글화 + 한국 스택 프리셋(Spring/NestJS/Next.js) |
| 온라인 강의 | ₩99,000 | "AI 에이전트 안전하게 부리기" |
| 기업 교육 | ₩3,000,000/일 | 설치 → 정책 설계 → 실습 워크샵 |
| 구축 컨설팅 | ₩10,000,000~ | 회사 맞춤 가드레일 설계 |
| 유료 프리셋 | ₩50,000/개 | 업종별 규칙팩(핀테크/의료/커머스) |

**시뮬레이션**: 강의 500명(₩49.5M) + 기업교육 월 2회(₩72M/년) + 컨설팅 연 5건(₩50M) ≈ **연 1.7억 원**

현금흐름이 가장 빠르다. **여기서 시작해 아이디어 1로 확장하는 경로를 권장.**

---

### 아이디어 4. React 실시간 승인 대시보드 ★★★★

**컨셉**: "AI가 막히면 → 폰으로 알림 → 엄지로 승인"

```
[푸시 알림]
Worker 승인 대기
작업: 결제 모듈 리팩터링
요청: git push origin main (R12)
영향: 12개 파일, +340 -128 / 리스크: 중간
[승인] [보류] [거부]
```

CCH에 `deferred-ops.jsonl`(보류 큐)과 `bin/harness deferred list|approve <id>`가 **이미 존재**한다.
그 위에 React + 모바일(PWA) 레이어만 얹으면 된다.

**스택**: Next.js 15 + React 19 + Tailwind + shadcn/ui / WebSocket 또는 SSE / PWA / Fastify 또는 Laravel

**가격**: 개인 $9/월, 팀(10명) $99/월, 셀프호스팅 $499 일회성

데모가 화려해 마케팅에 유리. **아이디어 1의 "얼굴"로 통합하면 최강.**

---

### 아이디어 5~8 (요약)

| # | 아이디어 | 수익 모델 | 난이도 | 추천 |
|---|---|---|---|---|
| 5 | 업종별 규칙팩 마켓플레이스 (R01–R16 프리셋 판매) | 개당 $19~99, 수수료 30% | ★★ | ★★★ |
| 6 | AI 코드 감사 대행 (리포트 발급) | 건당 ₩3,000,000 | ★★ | ★★★ |
| 7 | Managed Agent 호스팅 (24시간 자동 개발 대행) | ₩1,500,000/월/프로젝트 | ★★★★★ | ★★ |
| 8 | 콘텐츠·커뮤니티 (뉴스레터/유튜브/스폰서) | 스폰서 ₩1,000,000/회 | ★ | ★★★★ |

---

## 9. 추천 로드맵

```
[1단계] 0~3개월   한국어 포크 + 강의 + 콘텐츠        → 월 약 500만 원
[2단계] 3~6개월   React 대시보드 + 규칙팩 판매        → 연 약 1.7억 원
[3단계] 6~18개월  거버넌스 SaaS + 컴플라이언스 상품   → 연 5억 원+
```

### 당장 실행할 5가지

1. 포크에 한국어 README 추가 후 공개
2. "AI가 `rm -rf` 하려다 차단되는" 30초 데모 영상 제작·배포
3. 랜딩 페이지 1장(Next.js) + 대기자 이메일 수집
4. 개발자 커뮤니티 3곳에 한글화 공유
5. 반응이 유의미하면 강의 기획 착수

### 핵심 조언

> **도구를 팔지 말고 안심을 팔아라.**
>
> CCH는 무료다. 그러나 "우리 팀 AI가 사고 치지 않는다는 확신"은 돈을 주고 산다.
> 그리고 CCH가 가장 약한 지점이 UI다. React 역량이 있다면 조합이 매우 좋다 —
> Go 엔진은 그대로 빌려 쓰고, 사람이 보는 얼굴만 만들면 된다.

---

## 부록: 참고 링크

| 항목 | 링크 |
|---|---|
| **이 저장소 (포크)** | <https://github.com/bmshin94/claude-code-harness> |
| 원본 저장소 (업스트림) | <https://github.com/Chachamaru127/claude-code-harness> |
| harness-mem (선택 의존) | <https://github.com/Chachamaru127/harness-mem> |
| 라이선스 | [LICENSE.md](../../LICENSE.md) (MIT) |
| 프로젝트 가이드 | [CLAUDE.md](../../CLAUDE.md) |
| 제품 계약 | [spec.md](../../spec.md) |
| 작업 계약 | [Plans.md](../../Plans.md) |
| 변경 이력 | [CHANGELOG.md](../../CHANGELOG.md) |
| 호스트 호환 매트릭스 | [docs/tool-capability-matrix.md](../tool-capability-matrix.md) |
| 모델 라우팅 정책 | [docs/model-routing-policy.md](../model-routing-policy.md) |
| 방어층 영향 범위 규약 | [.claude/rules/defense-layer-blast-radius.md](../../.claude/rules/defense-layer-blast-radius.md) |

---

*본 문서는 2026-09-21 세션에서 저장소 전수조사를 통해 작성되었습니다.
수치(파일 수, 라인 수, 규칙 ID, 버전)는 조사 시점 `v5.15.0` 기준 실측값이며,
수익화 항목의 매출 수치는 가정에 기반한 추정치로 보장된 실적이 아닙니다.*
