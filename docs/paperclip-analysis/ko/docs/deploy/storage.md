---
title: 스토리지
summary: 로컬 디스크 vs S3 호환 스토리지
---

Paperclip은 업로드된 파일(이슈 첨부파일, 이미지)을 구성 가능한 스토리지 제공자를 사용하여 저장합니다.

## 로컬 디스크 (기본값)

파일은 다음 위치에 저장됩니다:

```
~/.paperclip/instances/default/data/storage
```

구성이 필요하지 않습니다. 로컬 개발 및 단일 머신 배포에 적합합니다.

## S3 호환 스토리지

프로덕션 또는 다중 노드 배포의 경우, S3 호환 오브젝트 스토리지(AWS S3, MinIO, Cloudflare R2 등)를 사용하세요.

CLI를 통해 구성합니다:

```sh
pnpm paperclipai configure --section storage
```

## 구성

| 제공자 | 적합한 용도 |
|----------|----------|
| `local_disk` | 로컬 개발, 단일 머신 배포 |
| `s3` | 프로덕션, 다중 노드, 클라우드 배포 |

스토리지 구성은 인스턴스 구성 파일에 저장됩니다:

```
~/.paperclip/instances/default/config.json
```
