---
title: "Docker 설치"
seoTitle: "Docker 설치"
seoDescription: "Docker 설치 가이드: 애플리케이션 구축, 테스트, 배포를 위해 Docker Desktop을 설치하고 사용합니다"
datePublished: Wed Jul 24 2024 19:13:14 GMT+0000 (Coordinated Universal Time)
cuid: clz082ryf000409jv6wjfazam
slug: docker-install
tags: docker

---

# Docker?

> 애플리케이션을 신속하게 구축, 테스트 및 배포할 수 있는 소프트웨어 플랫폼

* 소프트웨어를 **컨테이너**라는 표준화된 유닛으로 패키징
    
* 라이브러리, 시스템 도구, 코드, 런타임 등 소프트웨어를 실행하는 데 필요한 모든 것이 담김
    
* 환경에 구애받지 않고 애플리케이션을 신속하게 배포 및 확장 가능
    

# **Install Docker Desktop on Mac**

[https://docs.docker.com/desktop/install/mac-install/](https://docs.docker.com/desktop/install/mac-install/)

`$ softwareupdate --install-rosetta`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721842174146/d31b09e2-1c77-430b-a25a-3514725fcd52.png align="center")

`$ docker -v`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721843553758/b09b9033-910d-41f6-98cf-b0b61cb4cd55.png align="center")

`$ docker pull nginx` 이미지 다운로드

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721844426851/e24d6184-ef36-44e8-9c35-a84d50977161.png align="center")

```bash
$ docker run -d -p 80:80 --name my-nginx nginx 컨테이너 생성

# 1. d 옵션 : 컨테이너를 백그라운드에서 실행하는 detached 모드로 실행
# 2. p 80:80 옵션 : 호스트 머신의 80 포트를 컨테이너의 80 포트로 매핑하여 
#    컨테이너 내에서 실행 중인 NGINX 웹 서버에 접근
# 3. -name my-nginx 옵션 : 컨테이너에 "my-nginx"라는 이름을 할당
# 4. nginx 인자 : 컨테이너를 생성할 때 사용할 Docker 이미지의 이름
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721844757953/dbbc22ad-9f8f-4558-9899-d1a331e80aad.png align="center")

# **Get Docker Desktop**

> 도커 예제 이미지 실행 연습

[https://docs.docker.com/guides/getting-started/get-docker-desktop/](https://docs.docker.com/guides/getting-started/get-docker-desktop/)

`docker run -d -p 8080:80 docker/welcome-to-docker`

`docker/welcome-to-docker` 해당 도커 예제 이미지를 사용하여 컨테이너를 실행하면 컨테이너를 백그라운드 모드(`-d` 옵션)로 실행하고, 호스트의 8080 포트와 컨테이너의 80 포트를 연결(`-p 8080:80`)한다.

이미지 안에는 프로젝트의 코드, 필요한 모든 라이브러리, 서비스를 실행하기 위한 환경 설정, 그리고 애플리케이션을 실행하는 데 필요한 서버 환경(예: 톰캣, 노드JS, 아파치 등)까지 모두 포함될 수 있다. 이를 바탕으로 도커 컨테이너를 실행하면 해당 환경에서 애플리케이션을 즉시 시작하고 실행할 수 있다.

해당 이미지를 실행하면 웹 브라우저를 통해 [`localhost:8080`](http://localhost:8080)에 접속하여 간단한 웹 페이지를 볼 수 있다.

# Pushing a Docker Image to Docker Hub

`$ docker login`  
`$ docker image ls`

```bash
# docker image tag format 
# 태그 : 일반적으로 이미지의 버전을 구분하기 위해 사용
$ docker tag <로컬 이미지 이름>:<태그> <레지스트리 주소>/<이미지 이름>:<태그>
$ docker docker tag nginx myoxonee/my-nginx:1.0
```

```bash
# docker Registry format 
$ docker push <레지스트리 주소>/<이미지 이름>:<태그>
$ docker push myoxonee/my-nginx:1.0
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721847064156/ec36b3ed-f00e-456e-baba-9dcbea53ff0a.png align="center")

```bash
# Docker Registry pull format 
$ docker pull <레지스트리 주소>/<이미지 이름>:<태그>
 
# Docker Registry pull example
$ docker pull myoxonee/my-nginx:1.0
```

# Docker Image

`Dockerfile`(이미지를 빌드하는 데 필요한 지시어와 명령어를 포함)을 작성하여 Docker 이미지를 생성하고, `docker build` 명령을 사용하여 `Dockerfile`을 기반으로 로컬에서 이미지를 생성한다.

도커 이미지는 설치 애플리케이션과 유사한 개념을 가지고 있다. 설치 애플리케이션 패키지가 필요한 모든 파일, 설정, 종속성 등을 포함해 소프트웨어를 설치하고 실행할 수 있게 만드는 것 처럼, 도커 이미지도 애플리케이션 실행에 필요한 모든 파일, 설정, 라이브러리, 운영체제의 일부 등을 포함한다.

웹 애플리캐이션을 배포하는 대표적인 방법으로 Jenkins를 사용해 코드 변경사항을 감지하고 자동으로 빌드와 테스트를 수행한 후, Docker 이미지를 빌드하고 Docker 컨테이너를 자동으로 배포하는 방식이 있다. Jenkins의 자동화 능력과 Docker의 환경 일관성이 결합되어 더 빠르고 안정적인 소프트웨어 배포가 가능해진다.

#### 참고자료

[https://aws.amazon.com/ko/docker/](https://aws.amazon.com/ko/docker/)  
[https://adjh54.tistory.com/350](https://adjh54.tistory.com/350)