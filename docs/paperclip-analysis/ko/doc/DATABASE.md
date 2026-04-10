# 데이터베이스

Paperclip은 [Drizzle ORM](https://orm.drizzle.team/)을 통해 PostgreSQL을 사용합니다. 데이터베이스를 실행하는 방법은 가장 간단한 것부터 프로덕션급까지 세 가지가 있습니다.

## 1. 내장 PostgreSQL — 설정 불필요

`DATABASE_URL`을 설정하지 않으면 서버가 자동으로 내장 PostgreSQL 인스턴스를 시작하고 로컬 데이터 디렉토리를 관리합니다.

```sh
pnpm dev
```

이것으로 충분합니다. 처음 시작할 때 서버는:

1. 저장용 `~/.paperclip/instances/default/db/` 디렉토리를 생성합니다
2. `paperclip` 데이터베이스가 존재하는지 확인합니다
3. 빈 데이터베이스에 대해 자동으로 마이그레이션을 실행합니다
4. 요청 처리를 시작합니다

데이터는 `~/.paperclip/instances/default/db/`에 재시작 간에 유지됩니다. 로컬 개발 데이터를 초기화하려면 해당 디렉토리를 삭제하세요.

보류 중인 마이그레이션을 수동으로 적용해야 하는 경우 실행합니다:

```sh
pnpm db:migrate
```

`DATABASE_URL`이 설정되지 않은 경우, 이 명령은 현재 활성 Paperclip 설정/인스턴스의 내장 PostgreSQL 인스턴스를 대상으로 합니다.

이 모드는 로컬 개발 및 단일 명령 설치에 이상적입니다.

Docker 참고: Docker 퀵스타트 이미지도 기본적으로 내장 PostgreSQL을 사용합니다. 컨테이너 재시작 간에 DB 상태를 유지하려면 `/paperclip`을 영구 저장하세요(`doc/DOCKER.md` 참조).

## 2. 로컬 PostgreSQL (Docker)

로컬에서 전체 PostgreSQL 서버를 사용하려면 포함된 Docker Compose 설정을 사용합니다:

```sh
docker compose up -d
```

이렇게 하면 `localhost:5432`에서 PostgreSQL 17이 시작됩니다. 그런 다음 연결 문자열을 설정합니다:

```sh
cp .env.example .env
# .env에 이미 포함되어 있습니다:
# DATABASE_URL=postgres://paperclip:paperclip@localhost:5432/paperclip
```

마이그레이션을 실행하거나(마이그레이션 생성 문제가 수정된 후) `drizzle-kit push`를 사용합니다:

```sh
DATABASE_URL=postgres://paperclip:paperclip@localhost:5432/paperclip \
  npx drizzle-kit push
```

서버를 시작합니다:

```sh
pnpm dev
```

## 3. 호스팅 PostgreSQL (Supabase)

프로덕션에는 호스팅된 PostgreSQL 제공자를 사용합니다. [Supabase](https://supabase.com/)는 무료 티어가 있는 좋은 옵션입니다.

### 설정

1. [database.new](https://database.new)에서 프로젝트를 생성합니다
2. **Project Settings > Database > Connection string**으로 이동합니다
3. URI를 복사하고 비밀번호 플레이스홀더를 데이터베이스 비밀번호로 대체합니다

### 연결 문자열

Supabase는 두 가지 연결 모드를 제공합니다:

**직접 연결** (포트 5432) — 마이그레이션 및 일회성 스크립트에 사용:

```
postgres://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres
```

**Supavisor를 통한 연결 풀링** (포트 6543) — 애플리케이션에 사용:

```
postgres://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres
```

### 설정

`.env`에 `DATABASE_URL`을 설정합니다:

```sh
DATABASE_URL=postgres://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres
```

연결 풀링(포트 6543)을 사용하는 경우, `postgres` 클라이언트는 prepared statements를 비활성화해야 합니다. `packages/db/src/client.ts`를 업데이트합니다:

```ts
export function createDb(url: string) {
  const sql = postgres(url, { prepare: false });
  return drizzlePg(sql, { schema });
}
```

### 스키마 푸시

```sh
# 스키마 변경에는 직접 연결(포트 5432)을 사용합니다
DATABASE_URL=postgres://postgres.[PROJECT-REF]:[PASSWORD]@...5432/postgres \
  npx drizzle-kit push
```

### 무료 티어 제한

- 500 MB 데이터베이스 저장소
- 200 동시 연결
- 1주 동안 비활성 시 프로젝트 일시 중지

현재 세부사항은 [Supabase 가격](https://supabase.com/pricing)을 참조하세요.

## 모드 간 전환

데이터베이스 모드는 `DATABASE_URL`로 제어됩니다:

| `DATABASE_URL` | 모드 |
|---|---|
| 미설정 | 내장 PostgreSQL (`~/.paperclip/instances/default/db/`) |
| `postgres://...localhost...` | 로컬 Docker PostgreSQL |
| `postgres://...supabase.com...` | 호스팅 Supabase |

Drizzle 스키마(`packages/db/src/schema/`)는 모드에 관계없이 동일하게 유지됩니다.

## 비밀 저장소

Paperclip은 비밀 메타데이터와 버전을 다음에 저장합니다:

- `company_secrets`
- `company_secret_versions`

로컬/기본 설치의 경우, 활성 제공자는 `local_encrypted`입니다:

- 비밀 자료는 로컬 마스터 키로 저장 시 암호화됩니다.
- 기본 키 파일: `~/.paperclip/instances/default/secrets/master.key` (없으면 자동 생성).
- CLI 설정 위치: `~/.paperclip/instances/default/config.json`의 `secrets.localEncrypted.keyFilePath`.

선택적 오버라이드:

- `PAPERCLIP_SECRETS_MASTER_KEY` (base64, hex, 또는 원시 32자 문자열로 된 32바이트 키)
- `PAPERCLIP_SECRETS_MASTER_KEY_FILE` (커스텀 키 파일 경로)

새 인라인 민감 env 값을 차단하는 엄격 모드:

```sh
PAPERCLIP_SECRETS_STRICT_MODE=true
```

다음을 통해 엄격 모드와 제공자 기본값을 설정할 수 있습니다:

```sh
pnpm paperclipai configure --section secrets
```

인라인 비밀 마이그레이션 명령:

```sh
pnpm secrets:migrate-inline-env --apply
```
