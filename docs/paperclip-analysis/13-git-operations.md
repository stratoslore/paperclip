# 13. Git 운영 가이드 (Upstream 추적 + 별도 Git)

## 구조

```
[내 저장소] ←── origin (내 GitHub)
     │
     └── upstream (paperclipai/paperclip 공식)
```

| Remote | URL | 역할 |
|--------|-----|------|
| `upstream` | https://github.com/paperclipai/paperclip.git | 공식 오픈소스 추적 |
| `origin` | (내 GitHub 저장소 URL) | 커스텀 버전 관리 |

## 브랜치 전략

```
main ─── 내 커스텀 (i18n 등) 기반 안정 브랜치
  │
  ├── feature/* ─── 새 기능 개발
  │
  └── upstream-sync ─── upstream 업데이트 머지용
```

## 일상 운영 명령어

### upstream 최신 변경 가져오기

```bash
# upstream 최신 상태 fetch
git fetch upstream --tags

# 최신 릴리스 태그 확인
git tag -l "v*" | sort -V | tail -5

# upstream master와 내 main 비교
git log main..upstream/master --oneline | head -10
```

### upstream 업데이트 머지

```bash
# sync 브랜치에서 작업
git checkout -b upstream-sync

# upstream master를 머지 (충돌 시 수동 해결)
git merge upstream/master

# 충돌 해결 후
git checkout main
git merge upstream-sync
git branch -d upstream-sync
```

### 특정 버전으로 업그레이드

```bash
# 특정 태그 머지
git merge v2026.403.0

# 충돌 가능 파일 (i18n 관련):
# - ui/src/pages/*.tsx (새 페이지 추가 시)
# - ui/src/components/*.tsx (컴포넌트 변경 시)
# - package.json (의존성 변경 시)
```

### i18n 충돌 해결 가이드

upstream이 업데이트되면 주로 다음에서 충돌 발생:

1. **새 페이지/컴포넌트 추가**: `useTranslation` import + `t()` 적용 필요
2. **기존 파일 문자열 변경**: 번역 JSON 키 업데이트 필요
3. **패키지 의존성**: `package.json` 충돌 → 수동 머지

```bash
# 충돌 발생 시 확인
git diff --name-only --diff-filter=U

# i18n 관련 파일만 확인
git diff --name-only --diff-filter=U | grep -E "i18n|locales"
```

## 내 GitHub 저장소 설정

```bash
# 1. GitHub에 새 저장소 생성 후
git remote add origin https://github.com/<username>/agent-office.git

# 2. 최초 push
git push -u origin main

# 3. 이후 push
git push origin main
```

## Docker 패키징

### 로컬 이미지 빌드

```bash
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
docker compose -f docker/docker-compose.yml build
```

### 이미지 태깅 및 레지스트리 Push

```bash
# 빌드
docker build -t agent-office:latest .

# 태깅
docker tag agent-office:latest ghcr.io/<username>/agent-office:latest
docker tag agent-office:latest ghcr.io/<username>/agent-office:v0.3.1-i18n

# Push (GitHub Container Registry)
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin
docker push ghcr.io/<username>/agent-office:latest
docker push ghcr.io/<username>/agent-office:v0.3.1-i18n
```

### docker-compose.yml (프로덕션)

```yaml
services:
  db:
    image: postgres:17-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: paperclip
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: paperclip

  server:
    image: ghcr.io/<username>/agent-office:latest
    ports:
      - "3100:3100"
    environment:
      DATABASE_URL: postgres://paperclip:${DB_PASSWORD}@db:5432/paperclip
      BETTER_AUTH_SECRET: ${BETTER_AUTH_SECRET}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

## upstream 업데이트 주기 권장

| 빈도 | 내용 |
|------|------|
| 주 1회 | `git fetch upstream --tags` — 새 릴리스 확인 |
| 월 1회 | upstream 주요 변경사항 머지 검토 |
| 메이저 버전 | 전체 충돌 해결 + i18n 재검증 |
