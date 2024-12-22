---
title: "[도커] 명령어 정리"
description: "도커 명령어에 대해 알아보자"
date: 2024-12-22
update: 2024-12-22
tags:
  - 도커
  - 단순 지식
series: "Docker and Kubernetes"
---

## docker run

```bash
docker run 이미지_해시값
docker run 2ddf2wsd9fvg
```

- 이미지를 기반으로 새 컨테이너를 만들고 해당 컨테이너가 실행된다. attached 모드가 기본값이다.

```bash
docker run -p outbound포트:inbound포트 이미지_해시값
docker run -p 3000:80 2ddf2wsd9fvg
```

- 컨테이너의 인바운드/아웃바운드 포트를 설정하여 컨테이너와 통신할 수 있게 한다.

```bash
docker run -p outbound포트:inbound포트 -d 이미지_해시값
docker run -p 3000:80 -d 2ddf2wsd9fvg
```
- detached 모드로 컨테이너를 실행한다.

## docker start

```bash
docker start 컨테이너ID_혹은_이름
docker start 2ddf2wsd9fvg
```
- 중지된 컨테이너를 재시작한다. detached 모드가 기본값이다.

```bash
docker start -a 컨테이너ID_혹은_이름
docker start -a 2ddf2wsd9fvg
```
- 중지된 컨테이너를 attached 모드로 재시작한다.

## docker stop

```bash
docker stop 컨테이너ID_혹은_이름
docker stop exciting_apple
```
- 실행중인 컨테이너를 중지한다.

## docker attach
```bash
docker attach 컨테이너ID_혹은_이름
```
- 콘솔와 컨테이너를 연결한다.

## docker logs
```bash
docker logs 컨테이너ID_혹은_이름
docker logs exciting_apple
```
- 해당 컨테이너에 의해 출력된 과거의 로그를 보여준다.

```bash
docker logs -f 컨테이너ID_혹은_이름
docker logs -f exciting_apple
```
- 해당 컨테이너에 의해 출력된 과거의 로그를 보여준다. + 콘솔에서 해당 컨테이너의 로그를 수신 대기한다.(attach)

## docker ps

```bash
docker ps
```
- 현재 실행 중인 컨테이너 목록을 출력한다.

```bash
docker ps -a
```
- 모든 컨테이너 목록을 출력한다.

## docker build
```bash
docker build 도커파일_경로
docker build .
```
- dockerfile을 참조하여 이미지를 생성한다.
