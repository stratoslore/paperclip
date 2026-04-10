# 08. 설정 및 환경변수

> 소스: `packages/shared/src/config-schema.ts`, `server/src/config.ts`

## 설정 소스 우선순위

```
1. 환경변수 (PAPERCLIP_*, DATABASE_URL 등)  ← 최우선
2. 설정 파일 (~/.paperclip/instances/{id}/config.json)
3. .env 파일 (작업 디렉토리 또는 Paperclip 홈)
4. 하드코딩된 기본값                        ← 최하위
```

## 설정 파일 구조

경로: `~/.paperclip/instances/{instance-id}/config.json`

```json
{
  "$meta": {
    "version": 1,
    "updatedAt": "2026-04-03T00:00:00Z",
    "source": "onboard"
  },
  "database": {
    "mode": "embedded-postgres",
    "embeddedPostgresDataDir": "~/.paperclip/instances/default/db",
    "embeddedPostgresPort": 54329,
    "backup": {
      "enabled": true,
      "intervalMinutes": 60,
      "retentionDays": 30,
      "dir": "~/.paperclip/instances/default/data/backups"
    }
  },
  "server": {
    "deploymentMode": "local_trusted",
    "exposure": "private",
    "host": "127.0.0.1",
    "port": 3100,
    "allowedHostnames": [],
    "serveUi": true
  },
  "auth": {
    "baseUrlMode": "auto",
    "disableSignUp": false
  },
  "storage": {
    "provider": "local_disk",
    "localDisk": {
      "baseDir": "~/.paperclip/instances/default/data/storage"
    }
  },
  "secrets": {
    "provider": "local_encrypted",
    "strictMode": false,
    "localEncrypted": {
      "keyFilePath": "~/.paperclip/instances/default/secrets/master.key"
    }
  },
  "telemetry": {
    "enabled": true
  }
}
```

## 전체 환경변수 레퍼런스

### 서버 기본

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PORT` | `3100` | 서버 포트 |
| `HOST` | `127.0.0.1` | 바인드 호스트 |
| `SERVE_UI` | `true` | UI 정적 파일 서빙 |

### 배포 모드

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | `local_trusted` / `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` / `public` |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | `[]` | private 모드 허용 호스트 (쉼표 구분) |

### 데이터베이스

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `DATABASE_URL` | - | 외부 PostgreSQL 연결 문자열 |
| `PAPERCLIP_DB_BACKUP_ENABLED` | `true` | 자동 백업 활성화 |
| `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES` | `60` | 백업 간격 (분) |
| `PAPERCLIP_DB_BACKUP_RETENTION_DAYS` | `30` | 백업 보존 기간 (일) |
| `PAPERCLIP_DB_BACKUP_DIR` | - | 백업 디렉토리 |
| `PAPERCLIP_MIGRATION_AUTO_APPLY` | `false` | 자동 마이그레이션 |
| `PAPERCLIP_MIGRATION_PROMPT` | `true` | 마이그레이션 프롬프트 표시 |
| `PAPERCLIP_EMBEDDED_POSTGRES_VERBOSE` | `false` | 내장 PG 로그 상세 |

### 인증

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `BETTER_AUTH_SECRET` | - | 인증 시크릿 |
| `PAPERCLIP_AGENT_JWT_SECRET` | - | 에이전트 JWT 시크릿 |
| `BETTER_AUTH_TRUSTED_ORIGINS` | - | 신뢰할 수 있는 오리진 |
| `PAPERCLIP_AUTH_BASE_URL_MODE` | `auto` | `auto` / `explicit` |
| `PAPERCLIP_PUBLIC_URL` | - | 공개 base URL |
| `PAPERCLIP_AUTH_PUBLIC_BASE_URL` | - | 인증 공개 base URL |
| `PAPERCLIP_AUTH_DISABLE_SIGN_UP` | `false` | 회원가입 비활성화 |

### 하트비트

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `HEARTBEAT_SCHEDULER_ENABLED` | `true` | 하트비트 스케줄러 활성화 |
| `HEARTBEAT_SCHEDULER_INTERVAL_MS` | `30000` | 스케줄러 틱 간격 (ms) |

### 스토리지

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PAPERCLIP_STORAGE_PROVIDER` | `local_disk` | `local_disk` / `s3` |
| `PAPERCLIP_STORAGE_LOCAL_DIR` | - | 로컬 스토리지 디렉토리 |
| `PAPERCLIP_STORAGE_S3_BUCKET` | - | S3 버킷 |
| `PAPERCLIP_STORAGE_S3_REGION` | - | S3 리전 |
| `PAPERCLIP_STORAGE_S3_ENDPOINT` | - | S3 엔드포인트 (MinIO 등) |
| `PAPERCLIP_STORAGE_S3_PREFIX` | - | S3 키 접두사 |
| `PAPERCLIP_STORAGE_S3_FORCE_PATH_STYLE` | `false` | S3 경로 스타일 강제 |

### 시크릿

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PAPERCLIP_SECRETS_PROVIDER` | `local_encrypted` | 시크릿 프로바이더 |
| `PAPERCLIP_SECRETS_STRICT_MODE` | `false` | 엄격 모드 |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | - | 마스터 키 파일 경로 |

### 텔레메트리

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PAPERCLIP_TELEMETRY_DISABLED` | `false` | 텔레메트리 비활성화 |
| `DO_NOT_TRACK` | - | 표준 추적 거부 |

### 기타

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `PAPERCLIP_HOME` | `~/.paperclip` | Paperclip 홈 디렉토리 |
| `PAPERCLIP_INSTANCE_ID` | `default` | 인스턴스 ID |
| `PAPERCLIP_CONFIG` | - | 설정 파일 경로 오버라이드 |
| `PAPERCLIP_ENABLE_COMPANY_DELETION` | `false` | 회사 삭제 허용 |
| `PAPERCLIP_UI_DEV_MIDDLEWARE` | `false` | Vite 개발 미들웨어 |
| `PAPERCLIP_OPEN_ON_LISTEN` | `true` | 서버 시작 시 브라우저 열기 |

## Zod 스키마 상세

`packages/shared/src/config-schema.ts`에서 Zod로 설정 스키마를 정의한다:

### Database Config

```typescript
{
  mode: z.enum(['embedded-postgres', 'postgres']).default('embedded-postgres'),
  connectionString: z.string().optional(),  // mode='postgres' 시 필수
  embeddedPostgresDataDir: z.string().default('~/.paperclip/instances/default/db'),
  embeddedPostgresPort: z.number().default(54329),
  backup: {
    enabled: z.boolean().default(true),
    intervalMinutes: z.number().min(1).max(7200).default(60),
    retentionDays: z.number().min(1).max(3650).default(30),
    dir: z.string(),
  }
}
```

### Server Config

```typescript
{
  deploymentMode: z.enum(['local_trusted', 'authenticated']).default('local_trusted'),
  exposure: z.enum(['private', 'public']).default('private'),
  host: z.string().default('127.0.0.1'),
  port: z.number().min(1).max(65535).default(3100),
  allowedHostnames: z.array(z.string()).default([]),
  serveUi: z.boolean().default(true),
}
```

### Storage Config

```typescript
{
  provider: z.enum(['local_disk', 's3']).default('local_disk'),
  localDisk: {
    baseDir: z.string(),
  },
  s3: {
    bucket: z.string(),
    region: z.string(),
    endpoint: z.string().optional(),
    prefix: z.string(),
    forcePathStyle: z.boolean(),
  },
}
```

### Secrets Config

```typescript
{
  provider: z.enum([
    'local_encrypted',
    'aws_secrets_manager',
    'gcp_secret_manager',
    'vault'
  ]).default('local_encrypted'),
  strictMode: z.boolean().default(false),
  localEncrypted: {
    keyFilePath: z.string(),
  },
}
```

## 디렉토리 구조

```
~/.paperclip/
  └── instances/
      └── default/              # 기본 인스턴스
          ├── config.json       # 설정 파일
          ├── .env              # 시크릿 환경변수
          ├── db/               # 내장 PostgreSQL 데이터
          ├── data/
          │   ├── storage/      # 파일 스토리지
          │   └── backups/      # DB 백업
          ├── secrets/
          │   └── master.key    # 시크릿 암호화 키
          ├── logs/             # 로그 파일
          └── plugins/          # 플러그인 디렉토리
```
