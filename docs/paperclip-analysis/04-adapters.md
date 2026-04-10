# 04. 에이전트 어댑터 시스템

> 소스: `packages/adapters/`

Paperclip은 **어댑터 패턴**으로 다양한 AI 에이전트 런타임을 통합한다. "하트비트를 받을 수 있으면, 채용 가능하다."

## 지원 어댑터 목록

| 어댑터 | 타입 코드 | 설명 |
|--------|-----------|------|
| Claude Code | `claude_local` | Claude Code CLI 로컬 실행 |
| OpenAI Codex | `codex_local` | Codex CLI 로컬 실행 |
| Cursor | `cursor` | Cursor 에디터 로컬 실행 |
| Google Gemini | `gemini_local` | Gemini CLI 로컬 실행 |
| OpenClaw | `openclaw_gateway` | OpenClaw 원격 게이트웨이 |
| OpenCode | `opencode_local` | OpenCode 로컬 실행 |
| Pi | `pi_local` | Pi 로컬 실행 |

## 어댑터 인터페이스

모든 어댑터는 다음 인터페이스를 구현한다:

```typescript
interface Adapter {
  execute(config: ExecutionConfig): Promise<AdapterExecutionResult>;
}

interface AdapterExecutionResult {
  exitCode?: number;
  signal?: string;             // 'SIGTERM', 'SIGKILL'
  error?: string;
  stdout?: string;
  stderr?: string;
  resultJson?: Record<string, unknown>;
  usageSummary?: UsageSummary;
  sessionCodec?: AdapterSessionCodec;  // 세션 상태 (다음 하트비트용)
}

interface UsageSummary {
  inputTokens?: number;
  outputTokens?: number;
  cacheHitTokens?: number;
  totalTokens?: number;
  billedTokens?: number;
  costCents?: number;
}
```

## 어댑터 구조

각 어댑터는 3개 레이어로 구성된다:

```
packages/adapters/{adapter-name}/
  ├── src/
  │   ├── cli/      # CLI 통합 (설정 입력 프롬프트)
  │   ├── server/   # 서버 사이드 실행 로직
  │   └── ui/       # UI 설정 컴포넌트
  ├── package.json
  └── tsconfig.json
```

---

## Claude Code 어댑터 (claude_local)

> 소스: `packages/adapters/claude-local/src/`

### 지원 모델

| 모델 | ID |
|------|-----|
| Claude Opus 4.6 | `claude-opus-4-6` |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` |
| Claude Sonnet 4.5 | `claude-sonnet-4-5-20250929` |
| Claude Haiku 4.6 | `claude-haiku-4-6` |
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` |

### 설정 옵션

```typescript
{
  // 기본 설정
  cwd?: string;                          // 작업 디렉토리
  instructionsFilePath?: string;         // 마크다운 지시사항 파일
  model?: string;                        // Claude 모델 ID
  effort?: 'low' | 'medium' | 'high';   // 추론 노력도

  // 실행 제어
  chrome?: boolean;                      // --chrome 플래그
  promptTemplate?: string;               // 실행 프롬프트 오버라이드
  maxTurnsPerRun?: number;               // 최대 대화 턴 수
  dangerouslySkipPermissions?: boolean;  // 권한 검사 건너뛰기

  // 커스텀 실행
  command?: string;                      // 기본: 'claude'
  extraArgs?: string[];                  // 추가 CLI 인수
  env?: Record<string, string>;          // 커스텀 환경변수

  // 워크스페이스
  workspaceStrategy?: {
    type: 'git_worktree';
    baseRef?: string;
    branchTemplate?: string;
  };

  // 타임아웃
  timeoutSec?: number;                   // 실행 타임아웃
  graceSec?: number;                     // SIGTERM 유예 기간
}
```

### 런타임 환경변수 주입

실행 시 자동으로 주입되는 환경변수:

```bash
PAPERCLIP_AGENT_ID=<agent-uuid>
PAPERCLIP_COMPANY_ID=<company-uuid>
PAPERCLIP_API_URL=http://localhost:3100
PAPERCLIP_RUN_ID=<run-uuid>
PAPERCLIP_TASK_ID=<issue-uuid>          # (이슈 트리거 시)
PAPERCLIP_WAKE_REASON=issue_comment_mentioned  # (멘션 트리거 시)
PAPERCLIP_WAKE_COMMENT_ID=<comment-uuid>       # (멘션 트리거 시)
PAPERCLIP_API_KEY=<short-lived-jwt>
PAPERCLIP_WORKSPACE_*=...               # 워크스페이스 관련
PAPERCLIP_RUNTIME_*=...                 # 런타임 서비스 관련
```

### 세션 지원

Claude Code 어댑터는 하트비트 간 세션 유지를 지원한다:
- `agentTaskSessions` 테이블에 세션 코덱 저장
- 다음 하트비트에서 이전 세션 로드
- 세션 크기 초과 시 자동 압축

---

## Codex 어댑터 (codex_local)

> 소스: `packages/adapters/codex-local/src/`

### 지원 모델

| 모델 | ID |
|------|-----|
| GPT-5.4 | `gpt-5.4` |
| GPT-5.3 Codex | `gpt-5.3-codex` |
| GPT-5.3 Codex Spark | `gpt-5.3-codex-spark` |
| GPT-5 | `gpt-5` |
| O3 | `o3` |
| O4 Mini | `o4-mini` |
| O3 Mini | `o3-mini` |
| Codex Mini Latest | `codex-mini-latest` |

### 설정 옵션

```typescript
{
  cwd?: string;
  instructionsFilePath?: string;         // stdin 프롬프트에 프리펜드
  model?: string;
  modelReasoningEffort?: 'minimal' | 'low' | 'medium' | 'high' | 'xhigh';
  promptTemplate?: string;               // stdin 프롬프트 오버라이드
  search?: boolean;                      // --search 플래그
  dangerouslyBypassApprovalsAndSandbox?: boolean;
  command?: string;                      // 기본: 'codex'
  extraArgs?: string[];
  env?: Record<string, string>;
  workspaceStrategy?: { type: 'git_worktree'; ... };
  workspaceRuntime?: Record<string, unknown>;
  timeoutSec?: number;
  graceSec?: number;
}
```

### Claude vs Codex 차이점

| 항목 | Claude Local | Codex Local |
|------|-------------|-------------|
| 프롬프트 전달 | CLI 인수 | stdin 파이프 |
| 지시사항 | 런타임 파일 주입 | stdin에 프리펜드 |
| AGENTS.md | 선택 | 자동 적용 (억제 불가) |
| 홈 디렉토리 | 기본 | 회사별 CODEX_HOME 관리 |
| 스킬 주입 | SKILL.md 주입 | $CODEX_HOME/skills/에 자동 설치 |
| 추론 노력 | effort (low/medium/high) | modelReasoningEffort (minimal~xhigh) |

---

## OpenClaw Gateway 어댑터 (openclaw_gateway)

> 소스: `packages/adapters/openclaw-gateway/src/`

OpenClaw은 원격 게이트웨이 프로토콜을 통해 연결되는 유일한 어댑터:

- 로컬 프로세스가 아닌 원격 API 호출
- SSE(Server-Sent Events) 기반 통신
- 별도의 인증 토큰 필요

---

## 어댑터 추가 방법

새로운 에이전트 런타임을 추가하려면:

1. `packages/adapters/{name}/` 디렉토리 생성
2. `cli/`, `server/`, `ui/` 하위 디렉토리 구현
3. `AdapterExecutionResult` 인터페이스 구현
4. `server/src/adapters/`에 서버 사이드 통합 추가
5. `packages/shared/src/constants.ts`에 어댑터 타입 등록

### 어댑터 구현 체크리스트

- [ ] 프로세스 실행 및 결과 캡처
- [ ] 환경변수 주입
- [ ] 스킬 주입 메커니즘
- [ ] 세션 관리 (선택)
- [ ] 타임아웃/그레이스 기간 처리
- [ ] 사용량(토큰, 비용) 파싱
- [ ] CLI 설정 프롬프트
- [ ] UI 설정 컴포넌트
