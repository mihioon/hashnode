---
title: "캐리지 리턴(cr)과 라인 피드(lf)"
seoTitle: "CRLF와 LF: 차이점과 사용처"
seoDescription: "Learn about CRLF (Windows) vs. LF (Unix) line breaks and their efficiency and compatibility benefits across operating systems"
datePublished: Tue Jul 30 2024 12:05:57 GMT+0000 (Coordinated Universal Time)
cuid: clz8dgekf000e09lfhkrk05r8
slug: cr-lf
tags: lf-crlf

---

# 개행(줄바꿈)문자

> `\r` Carriage Return(CR) 맨앞으로 이동하라는 뜻

> `\n` Line Feed(LF) New Line, 즉 새로운 라인이라는 뜻

타자기가 타이핑을 하는 방식에서 유래되었다.

![](https://velog.velcdn.com/images/myo_nee/post/9d90c39e-9886-43a3-8f03-3bef0962b6aa/image.png align="left")

## Windows : CRLF

타자기가 한 줄을 모두 입력한 후 다음 줄을 입력하기 위해 한 줄 간격만큼 아래로 이동하는 것을 LF(Line Feed),

그리고 한 줄을 모두 입력해서 잉크를 찍는 부분이 종이의 가장 오른쪽에 위치해있으니 다시 처음부터 입력하기 위해 가장 왼쪽으로 이동하는 것을 Carriage Return(CR) 이라고 한다.

초창기에는 컴퓨터가 아닌 teletype이라는 타이핑용 장비를 직접 제어하여 코드를 입력했기 때문에 CR과 LF가 모두 필요했고, Windows는 바로 이 방식을 참고해 CRLF방식을 채택했다. 실제로 window에서 작성된 파일을 확인해보면 CRLF방식으로 저장된 것을 확인할 수 있다.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><code>Notepad</code>에서 <code>보기&gt;기호보기&gt;줄 끝 표시</code> 로 간단하게 확인이 가능함.</div>
</div>

![](https://velog.velcdn.com/images/myo_nee/post/46ad4408-3e31-4909-9e4f-144a04ffec8e/image.png align="left")

## Linux와 최신 MacOS : LF

하지만 점차 디지털화 되어가면서 화면(display)를 사용하게 되자, 커서를 맨 앞으로 되돌리지(CR) 않아도 충분히 줄을 바꾸겠다는 의미를 전달할 수 있게 되었다. 굳이 한 문자(1Byte)로 표현할 수 있는 정보를 두 개의 문자(2Byte)로 늘려서 표현하지 않아도 되는 것이다.

그래서 Windows와 달리 Unix는 LF방식을, 초창기 클래식 MacOS는 CR방식을 채택했다. Linux는 Unix의 방식을 그대로 가져왔고, MacOS의 경우는 최신 운영체제에선 LF방식으로 변경되었다.

## OS별로 줄바꿈 문자에 차이가 생겨서

인식하는 문자가 다르기 때문에 같은 파일이 OS에 따라 다르게 보일 수 있는 문제가 발생했다. 로컬과 서버의 OS가 다른 경우나, 다른 개발자들과 함께 작업하는 환경에서 OS에 차이가 있을 경우 혼란이 발생할 수 있다.

그래서 오늘날 이런 문제를 방지하기 위해 작업자들끼리 규칙(e.g. 코딩 컨벤션)을 정하거나, 다양한 방법으로 OS가 달라져도 파일의 양식을 맞출 수 있도록(e.g. git, Docker) 설정해주고 있다.

## LF방식의 장점

일반적으로 LF를 추천하는데, 아무래도 CRLF에 비해 불필요한 문자가 하나 줄어든다는 점이 큰 이점이다. 파일 크기는 작을수록 좋으니까.

여러 서버 환경에서 Linux 운영 체제가 주로 사용되고 있기도 하고, Git과 같은 형상 관리 도구도 LF 방식을 표준으로 채택하고 있기 때문에 호환성과 파일 처리 효율성도 좋다.

---

#### 참고 자료

* [https://m.blog.naver.com/taeil34/221325864981](https://m.blog.naver.com/taeil34/221325864981)
    
* [https://en.wikipedia.org/wiki/Newline](https://en.wikipedia.org/wiki/Newline)