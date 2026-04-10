---
title: 환경 변수
summary: 전체 환경 변수 참조
---

Paperclip이 서버 구성에 사용하는 모든 환경 변수입니다.

## 서버 구성

| 변수 | 기본값 | 설명 |
|----------|---------|-------------|
| `PORT` | `3100` | 서버 포트 |
| `HOST` | `127.0.0.1` | 서버 호스트 바인딩 |
| `DATABASE_URL` | (임베디드) | PostgreSQL 연결 문자열 |
| `PAPERCLIP_HOME` | `~/.paperclip` | 모든 Paperclip 데이터의 기본 디렉토리 |
| `PAPERCLIP_INSTANCE_ID` | `default` | 인스턴스 식별자 (여러 로컬 인스턴스용) |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | 런타임 모드 재정의 |

## 시크릿

| 변수 | 기본값 | 설명 |
|----------|---------|-------------|
| `PAPERCLIP_SECRETS_MASTER_KEY` | (파일에서) | 32바이트 암호화 키 (base64/hex/raw) |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | `~/.paperclip/.../secrets/master.key` | 키 파일 경로 |
| `PAPERCLIP_SECRETS_STRICT_MODE` | `false` | 민감한 환경 변수에 시크릿 참조 필수 |

## 에이전트 런타임 (에이전트 프로세스에 주입)

에이전트를 호출할 때 서버가 자동으로 설정합니다:

| 변수 | 설명 |
|----------|-------------|
| `PAPERCLIP_AGENT_ID` | 에이전트의 고유 ID |
| `PAPERCLIP_COMPANY_ID` | 회사 ID |
| `PAPERCLIP_API_URL` | Paperclip API 기본 URL |
| `PAPERCLIP_API_KEY` | API 인증을 위한 단기 JWT |
| `PAPERCLIP_RUN_ID` | 현재 하트비트 실행 ID |
| `PAPERCLIP_TASK_ID` | 이 깨어남을 트리거한 이슈 |
| `PAPERCLIP_WAKE_REASON` | 깨어남 트리거 이유 |
| `PAPERCLIP_WAKE_COMMENT_ID` | 이 깨어남을 트리거한 코멘트 |
| `PAPERCLIP_APPROVAL_ID` | 처리된 승인 ID |
| `PAPERCLIP_APPROVAL_STATUS` | 승인 결정 |
| `PAPERCLIP_LINKED_ISSUE_IDS` | 쉼표 구분 연결된 이슈 ID |

## LLM 제공자 키 (어댑터용)

| 변수 | 설명 |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Anthropic API 키 (Claude Local 어댑터용) |
| `OPENAI_API_KEY` | OpenAI API 키 (Codex Local 어댑터용) |
