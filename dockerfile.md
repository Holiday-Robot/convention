# Dockerfile *Draft*

## 커뮤니티 표준

이 가이드는 아래 문서를 기반으로 합니다. 아래에서 명시적으로 언급하지 않은 사항은 이 문서를 따릅니다.

- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)


## 도구

### Formatter


### Linter



## 스타일 규칙

린터가 강제할 수 있는 규칙만을 정의합니다.

### 베이스 이미지 태그

베이스 이미지는 반드시 명시적인 태그 또는 digest를 지정합니다. `latest`는 사용하지 않습니다.

```dockerfile
# ❌ Bad
FROM ubuntu

# ❌ Bad
FROM ubuntu:latest

# 🟢 Good
FROM ubuntu:24.04
```

재현 가능한 빌드를 보장하려면 digest를 사용합니다.

```dockerfile
FROM ubuntu:24.04@sha256:...
```

### `RUN` 명령어 결합

하나의 논리적 작업은 `&&`로 연결하여 단일 `RUN` 명령어로 작성합니다.
레이어 수를 줄이고, 캐시를 불필요하게 분산시키지 않습니다.

```dockerfile
# ❌ Bad
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# 🟢 Good
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
	--mount=type=cache,target=/var/lib/apt,sharing=locked \
	apt-get update \
	&& apt-get install --yes --no-install-recommends \
		curl
```

`apt-get update`와 `apt-get install`은 항상 같은 `RUN`에 작성합니다.
별도 레이어로 분리하면 캐시된 `update` 결과가 stale해질 수 있습니다.

연결 연산자는 줄의 마지막이 아니라 개행 후 다음 줄의 시작에 위치해야 합니다.
줄 끝에 위치하면, 줄이 삭제되거나 주석 처리될 때 연결 연산자가 남아있어 휴먼 에러를 유발할 수 있습니다.



## 설계 지침

도구로 강제할 수 없지만, 이미지의 일관성과 보안을 위해 따라야 할 지침입니다.

### 멀티스테이지 빌드

빌드 도구와 런타임 환경을 분리하여 최종 이미지 크기를 최소화합니다.

```dockerfile
FROM golang:1.24 AS builder
WORKDIR /src
COPY . .
RUN go build -o /bin/app .

FROM scratch
COPY --from=builder /bin/app /app
ENTRYPOINT ["/app"]
```

### `.dockerignore`

빌드 컨텍스트에 불필요한 파일이 포함되지 않도록 `.dockerignore`를 관리합니다.
`.git`, 로컬 빌드 산출물, 시크릿 파일 등을 제외합니다.

```
.git
*.md
dist/
.env
```
