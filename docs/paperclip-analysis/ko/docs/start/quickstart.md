---
title: 빠른 시작
summary: 몇 분 안에 Paperclip을 실행하세요
---

5분 이내에 로컬에서 Paperclip을 실행하세요.

## 빠른 시작 (권장)

```sh
npx paperclipai onboard --yes
```

이 명령은 설정 과정을 안내하고, 환경을 구성하고, Paperclip을 실행합니다.

이미 Paperclip이 설치되어 있다면, `onboard`를 다시 실행해도 현재 설정과 데이터 경로가 유지됩니다. 설정을 편집하려면 `paperclipai configure`를 사용하세요.

나중에 Paperclip을 다시 시작하려면:

```sh
npx paperclipai run
```

> **참고:** 설정에 `npx`를 사용했다면, 명령어 실행 시 항상 `npx paperclipai`를 사용하세요. `pnpm paperclipai` 형식은 Paperclip 저장소의 클론된 복사본 내부에서만 작동합니다 (아래 로컬 개발 참조).

## 로컬 개발

Paperclip 자체에 기여하는 개발자를 위한 섹션입니다. 필수 조건: Node.js 20+ 및 pnpm 9+.

저장소를 클론한 후:

```sh
pnpm install
pnpm dev
```

이 명령은 API 서버와 UI를 [http://localhost:3100](http://localhost:3100)에서 시작합니다.

외부 데이터베이스가 필요하지 않습니다 — Paperclip은 기본적으로 임베디드 PostgreSQL 인스턴스를 사용합니다.

클론된 저장소에서 작업할 때 다음도 사용할 수 있습니다:

```sh
pnpm paperclipai run
```

이 명령은 설정이 없으면 자동으로 온보딩하고, 자동 복구 기능이 있는 상태 확인을 실행하고, 서버를 시작합니다.

## 다음 단계

Paperclip이 실행되면:

1. 웹 UI에서 첫 번째 회사를 생성합니다
2. 회사 목표를 정의합니다
3. CEO 에이전트를 생성하고 Adapter를 구성합니다
4. 더 많은 에이전트로 조직도를 구성합니다
5. 예산을 설정하고 초기 태스크를 할당합니다
6. 시작 — 에이전트들이 Heartbeat을 시작하고 회사가 운영됩니다

<Card title="핵심 개념" href="/start/core-concepts">
  Paperclip의 핵심 개념을 알아보세요
</Card>
