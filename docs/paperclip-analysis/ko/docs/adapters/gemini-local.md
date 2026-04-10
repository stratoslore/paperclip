---
title: Gemini Local
summary: Gemini CLI 로컬 어댑터 설정 및 구성
---

`gemini_local` 어댑터는 Google의 Gemini CLI를 로컬에서 실행합니다. `--resume`을 통한 세션 지속성, 스킬 주입, 구조화된 `stream-json` 출력 파싱을 지원합니다.

## 사전 요구 사항

- Gemini CLI 설치 (`gemini` 명령어 사용 가능)
- `GEMINI_API_KEY` 또는 `GOOGLE_API_KEY` 설정, 또는 로컬 Gemini CLI 인증 구성

## 구성 필드

| 필드 | 타입 | 필수 | 설명 |
|-------|------|----------|-------------|
| `cwd` | string | 예 | 에이전트 프로세스의 작업 디렉토리 (절대 경로; 권한이 허용되면 없을 시 자동 생성) |
| `model` | string | 아니오 | 사용할 Gemini 모델. 기본값은 `auto`. |
| `promptTemplate` | string | 아니오 | 모든 실행에 사용되는 프롬프트 |
| `instructionsFilePath` | string | 아니오 | 프롬프트 앞에 추가되는 마크다운 지시사항 파일 |
| `env` | object | 아니오 | 환경 변수 (시크릿 참조 지원) |
| `timeoutSec` | number | 아니오 | 프로세스 타임아웃 (0 = 타임아웃 없음) |
| `graceSec` | number | 아니오 | 강제 종료 전 유예 기간 |
| `yolo` | boolean | 아니오 | 무인 운영을 위해 `--approval-mode yolo` 전달 |

## 세션 지속성

어댑터는 하트비트 간에 Gemini 세션 ID를 유지합니다. 다음 깨어남 시 `--resume`으로 기존 대화를 이어서 에이전트가 컨텍스트를 유지합니다.

세션 재개는 cwd를 인식합니다: 마지막 실행 이후 작업 디렉토리가 변경된 경우, 새 세션이 시작됩니다.

알 수 없는 세션 오류로 재개에 실패하면, 어댑터는 자동으로 새 세션으로 재시도합니다.

## 스킬 주입

어댑터는 Paperclip 스킬을 Gemini 글로벌 스킬 디렉토리(`~/.gemini/skills`)에 심볼릭 링크합니다. 기존 사용자 스킬은 덮어쓰지 않습니다.

## 환경 테스트

UI의 "Test Environment" 버튼을 사용하여 어댑터 구성을 검증합니다. 다음을 확인합니다:

- Gemini CLI가 설치되어 있고 접근 가능한지
- 작업 디렉토리가 절대 경로이고 사용 가능한지 (권한이 허용되면 없을 시 자동 생성)
- API 키/인증 힌트 (`GEMINI_API_KEY` 또는 `GOOGLE_API_KEY`)
- CLI 준비 상태를 확인하기 위한 실시간 hello 프로브 (`gemini --output-format json "Respond with hello."`)
