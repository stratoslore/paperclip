이 체크리스트를 그대로 사용하세요.

1. 인증 모드로 Paperclip을 시작합니다.
```bash
cd <paperclip-repo-root>
pnpm dev --tailscale-auth
```
그런 다음 확인합니다:
```bash
curl -sS http://127.0.0.1:3100/api/health | jq
```

2. 클린/기본 OpenClaw Docker를 시작합니다.
```bash
OPENCLAW_RESET_STATE=1 OPENCLAW_BUILD=1 ./scripts/smoke/openclaw-docker-ui.sh
```
브라우저에서 출력된 `Dashboard URL` (`#token=...` 포함)을 엽니다.

3. Paperclip UI에서 `http://127.0.0.1:3100/CLA/company/settings`로 이동합니다.

4. OpenClaw 초대 프롬프트 흐름을 사용합니다.
- 초대 섹션에서 `Generate OpenClaw Invite Prompt`를 클릭합니다.
- `OpenClaw Invite Prompt`에서 생성된 프롬프트를 복사합니다.
- OpenClaw 메인 채팅에 하나의 메시지로 붙여넣습니다.
- 멈추면 후속 메시지를 하나 보냅니다: `How is onboarding going? Continue setup now.`

보안/제어 참고:
- OpenClaw 초대 프롬프트는 제어된 엔드포인트에서 생성됩니다:
  - `POST /api/companies/{companyId}/openclaw/invite-prompt`
  - 초대 권한이 있는 보드 사용자가 호출할 수 있습니다
  - 에이전트 호출자는 회사 CEO 에이전트로 제한됩니다

5. Paperclip UI에서 가입 요청을 승인한 다음, OpenClaw 에이전트가 CLA 에이전트에 나타나는지 확인합니다.

6. 게이트웨이 사전 점검 (작업 테스트 전 필수).
- 생성된 에이전트가 `openclaw_gateway` (`openclaw`가 아님)를 사용하는지 확인합니다.
- 게이트웨이 URL이 `ws://...` 또는 `wss://...`인지 확인합니다.
- 게이트웨이 토큰이 의미 있는 값인지 (빈 값이 아니고 / 1자 플레이스홀더가 아닌지) 확인합니다.
- OpenClaw Gateway 어댑터 UI는 일반 온보딩에서 `disableDeviceAuth`를 노출하지 않아야 합니다.
- 페어링 모드가 명시적인지 확인합니다:
  - 필수 기본값: 디바이스 인증 활성화 (`adapterConfig.disableDeviceAuth` false/미설정) 및 지속된 `adapterConfig.devicePrivateKeyPem`
  - 일반 온보딩에서 `disableDeviceAuth`에 의존하지 마세요
- 보드 인증으로 API 확인을 실행할 수 있는 경우:
```bash
AGENT_ID="<newly-created-agent-id>"
curl -sS -H "Cookie: $PAPERCLIP_COOKIE" "http://127.0.0.1:3100/api/agents/$AGENT_ID" | jq '{adapterType,adapterConfig:{url:.adapterConfig.url,tokenLen:(.adapterConfig.headers["x-openclaw-token"] // .adapterConfig.headers["x-openclaw-auth"] // "" | length),disableDeviceAuth:(.adapterConfig.disableDeviceAuth // false),hasDeviceKey:(.adapterConfig.devicePrivateKeyPem // "" | length > 0)}}'
```
- 예상 결과: `adapterType=openclaw_gateway`, `tokenLen >= 16`, `hasDeviceKey=true`, `disableDeviceAuth=false`.

페어링 핸드셰이크 참고:
- 클린 실행 기대: 첫 번째 작업은 수동 페어링 명령 없이 성공해야 합니다.
- 어댑터는 첫 `pairing required`에서 자동 페어링 승인 + 재시도를 한 번 시도합니다 (공유 게이트웨이 인증 토큰/비밀번호가 유효한 경우).
- 자동 페어링을 완료할 수 없는 경우 (예: 토큰 불일치 또는 대기 중인 요청 없음), 첫 번째 게이트웨이 실행은 여전히 `pairing required`를 반환할 수 있습니다.
- 이것은 Paperclip 초대 승인과는 별도의 승인입니다. OpenClaw 자체에서 대기 중인 디바이스를 승인해야 합니다.
- OpenClaw에서 승인한 다음 작업을 재시도하세요.
- 로컬 Docker 스모크 테스트의 경우, 호스트에서 승인할 수 있습니다:
```bash
docker exec openclaw-docker-openclaw-gateway-1 sh -lc 'openclaw devices approve --latest --json --url "ws://127.0.0.1:18789" --token "$(node -p \"require(process.env.HOME+\\\"/.openclaw/openclaw.json\\\").gateway.auth.token\")"'
```
- 대기 중인 디바이스와 페어링된 디바이스를 검사할 수 있습니다:
```bash
docker exec openclaw-docker-openclaw-gateway-1 sh -lc 'TOK="$(node -e \"const fs=require(\\\"fs\\\");const c=JSON.parse(fs.readFileSync(\\\"/home/node/.openclaw/openclaw.json\\\",\\\"utf8\\\"));process.stdout.write(c.gateway?.auth?.token||\\\"\\\");\")\"; openclaw devices list --json --url \"ws://127.0.0.1:18789\" --token \"$TOK\"'
```

7. 케이스 A (수동 이슈 테스트).
- OpenClaw 에이전트에 배정된 이슈를 생성합니다.
- 지침을 입력합니다: "post comment `OPENCLAW_CASE_A_OK_<timestamp>` and mark done."
- UI에서 확인: 이슈 상태가 `done`이 되고 댓글이 존재합니다.

8. 케이스 B (메시지 도구 테스트).
- OpenClaw에 배정된 다른 이슈를 생성합니다.
- 지침: "send `OPENCLAW_CASE_B_OK_<timestamp>` to main webchat via message tool, then comment same marker on issue, then mark done."
- 두 가지 모두 확인:
  - 이슈의 마커 댓글
  - OpenClaw 메인 채팅에 마커 텍스트 표시

9. 케이스 C (새 세션 메모리/스킬 테스트).
- OpenClaw에서 `/new` 세션을 시작합니다.
- 고유한 제목 `OPENCLAW_CASE_C_CREATED_<timestamp>`으로 Paperclip에 새 CLA 이슈를 생성하도록 요청합니다.
- Paperclip UI에서 새 이슈가 존재하는지 확인합니다.

10. 테스트 중 로그 확인 (선택 사항이지만 도움이 됨):
```bash
docker compose -f /tmp/openclaw-docker/docker-compose.yml -f /tmp/openclaw-docker/.paperclip-openclaw.override.yml logs -f openclaw-gateway
```

11. 예상 통과 기준.
- 사전 점검: `openclaw_gateway` + 플레이스홀더가 아닌 토큰 (`tokenLen >= 16`).
- 페어링 모드: 디바이스 인증이 활성화된 상태에서 안정적인 `devicePrivateKeyPem` 구성 (기본 경로).
- 케이스 A: `done` + 마커 댓글.
- 케이스 B: `done` + 마커 댓글 + 메인 채팅 메시지 표시.
- 케이스 C: 원래 작업 완료 및 `/new` 세션에서 새 이슈 생성.

원하시면 UI에서 동일한 단계를 실시간으로 보면서 기본 스모크 하네스를 실행하는 단일 "관찰자 모드" 명령어도 제공할 수 있습니다.
