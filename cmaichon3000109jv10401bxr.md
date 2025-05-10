---
title: "Ui 공통 모듈...을 만들게 됐어요"
seoTitle: "UI Common Module"
seoDescription: "Learn how to create a common UI module for mobile and desktop clients, streamlining design processes and enhancing communication between modules"
datePublished: Sat May 10 2025 14:53:25 GMT+0000 (Coordinated Universal Time)
cuid: cmaichon3000109jv10401bxr
slug: chat-ui-module
tags: modules, native, electron, contextbridge

---

우당탕탕 프론트 일기 1탄…

현재 서비스중인 프로그램엔 모바일 클라이언트와 윈도우 클라이언트가 각각 존재한다. 근데 두 클라이언트가 별도로 개발돼있어서 디자인 작업을 두 번 해야 하는 번거로움이 있어 이번에 일부 기능을 합치기로 했다. 채팅 기능인데, 덕분에 기존 채팅관련 소스를 파악하는 중이라 재밌고 신기해하는 중. 채팅 기능을 첨 접해봐가지구…헷

모듈 내부에 큰 기능이 있는 건 아니고 정말 UI만 지원하는 거라 역할을 분리해서 생각하는 게 중요하다. (서버 통신이나 데이터 저장은 각 클라이언트의 역할이고, 그나마 좀 중요한 기능이라고 하면 UI 상의 숫자계산정도? 이건 각 클라이언트에서 두 번 계산식 작성하느니 모듈에서 작업하는 게 맞을 것 같다.)

그래서 각 클라이언트에서 공통으로 호출할 인터페이스를 작성해야하는데, 이를 위해선 각 클라이언트에서 공통 모듈에 어떻게 접근 가능한지 알아두는 게 도움이 될 거다. 어떻게 연결하는 지를 알아야 다리를 만들어 줄 수 있으니깐…!

## 🍀 지원 클라이언트

* **Desktop (Windows)**: Electron (Chromium 기반 데스크탑 앱)
    
* **Mobile (Android)**: Native Android (Java)
    

## 0\. **모듈 구조**

* `chat.html`
    
* `chat.js`
    

모듈에는 두 파일이 있다고 일단 가정하자.  
그리고 다음의 두 기능을 예시로 들기로 했다.

1. `메시지 전송(모듈 → 클라이언트)`
    
2. `메시지 수신(클라이언트 → 모듈)`
    

주고받는 데이터 객체의 명칭은 `message`로 통일한다.

# 1\. **Desktop (Windows)** : Electron (Chromium 기반 데스크탑 앱)

참고로 PC client는 어차피 내가 작업해야한다.  
왜냐면 이 쪽 개발도 내가 담당하고있기 때문이다. (나는 나와 통신한다…)

설명에 앞서, 이 내용을 이해하기 위해서는 Electron 환경에서의 **메인 프로세스**와 **렌더러 프로세스**의 개념이 필요하다.

* **메인 프로세스**?  
    Electron 애플리케이션의 시작점이며, 창 생성, 이벤트 처리, 시스템 접근 등 앱의 핵심 로직이 실행
    
    * 즉, 채팅 기능에서 서버 통신하는 등의 동작을 담당
        
* **렌더러 프로세스**?  
    각각의 UI 창에서 실행되는 HTML/JS 환경으로, 실제 사용자 인터페이스를 담당  
    이번에 작업할 모듈(chat.js 포함)은 이 렌더러 프로세스 위에서 실행 됨
    
    * 즉, 채팅 기능의 UI 인터페이스를 담당
        

따라서, 채팅 기능을 모듈로 구성하려면 이 두 프로세스 간의 통신이 필요하다.  
이 통신을 **안전하게 연결해주는 브릿지**가 무엇인지 알아보자는 거다!

## 1) 모듈 → Main Process통신

### 브릿지 &gt; preload.js

> 💡 **preload.js?**  
> Electron에서 웹 페이지(렌더러)와 Node.js 기능(메인 프로세스/IPC) 사이를 안전하게 연결해주는 스크립트  
> 앱이 로드되기 전에 실행되며, 필요한 기능만 노출해주는 브릿지 역할

Electron 환경에서 가장 유의해야할 점은 보안이다. 기본 보안 설정에서는 HTML파일만 로드해선 내부에 선언된 JS 파일을 실행시킬 수 없다. 따라서 필요한 함수는 미리 **preload 스크립트**를 통해 명시적으로 등록해줘야한다.

다음은 메인 프로세스에 ChatInterface 라는 이름의 전역 객체를 window에 등록하는 작업이고, 이 ChatInterface를 통해 chat.js에서 Electron 메인 프로세스로 이벤트를 보낼 수 있다. 야호!

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('ChatInterface', {
  sendMessage: (message) => ipcRenderer.send('send-message', message)
});
```

### 모듈 &gt; chat.js

모듈에선 브릿지를 통해 메인 프로세스에 연결된 함수를 호출할 수 있다. 아직 이 코드만 봐선 감이 안 올수도 있지만, 마지막에 모바일과 같이 정리된 모듈의 인터페이스를 보면 바로 이해가 갈 거다.

```javascript
window.ChatInterface.sendMessage(message);
```

### Main Process &gt; main.js

반대로, **메인 프로세스**에서는 모듈(chat.js)이 브릿지를 통해 전송한 메시지를 수신하고 처리하면 된다.

```javascript
const { ipcMain } = require('electron');

ipcMain.on('send-message', (event, message) => {
  // 서버 통신 작업
});
```

그럼 이걸 왜 하냐?

이 쯤에서 **메시지 전송 처리 플로우**를 정리해보면 다음과 같다.

### 메시지 전송(발신) **플로우**

\[User\] → \[Chat UI(chat.html)\] → \[Renderer JS(chat.js)\] → \[Preload Bridge\] → \[Main Process\] → \[Server\]

```scss
[UI] ← 사용자가 메시지 입력 후 전송 버튼 클릭
   │
   ▼
[모듈 (chat.js)]
   └─ window.ChatInterface.sendMessage(message)
        │
        ▼
[preload.js]
   └─ ipcRenderer.send('send-message', message)
        │
        ▼
[Electron Main Process]
   └─ 서버에 메시지 전송
        │
        ▼
[서버]
```

다시 말하지만 모듈은 직접 서버와 통신해서는 안 된다! 그렇기 때문에 '메시지를 전송해'라고 메인 프로세스에 전달하는 과정이 필요하다.

Electron에서는 이 렌더러 ↔ 메인 간 통신을 위해 **IPC (Inter-Process Communication)** 메커니즘을 제공하며,  
이를 통해 렌더러(chat.js)는 **메시지를 메인에 안전하게 전달**하고, 메인은 **실제 서버와의 통신**을 담당하게 된다.

## 2) Electron Main Process → 모듈 통신

그럼 반대로 메인에서는 모듈로 어떻게 이벤트를 전송하냐?  
이번엔 먼저 플로우를 확인해보자. 전송 과정과 정확히 반대다.

### 메시지 수신 **플로우**

\[Server\] → \[Main Process\] → \[Preload Bridge\] → \[Renderer JS(chat.js)\] → \[Chat UI(chat.html)\] → \[User\]

```scss
[서버]
  └─ 메시지 수신 이벤트 발행
       │
       ▼
[Electron Main Process]
  └─ 서버로부터 메시지를 수신
  └─ 렌더러 프로세스에 전달: win.webContents.send('message-received', message)
       │
       ▼
[preload.js]
  └─ ipcRenderer.on('message-received', callback)
  └─ window.ChatInterface.onMessageReceived(message) 실행
       │
       ▼
[모듈 (chat.js)]
  └─ 전달받은 메시지를 UI에 렌더링 (예: displayMessage(message))
       │
       ▼
[UI]
  └─ 메시지가 표시됨
```

여기까진 좋다.  
근데 메인 → 모듈 통신은 콜백 함수가 들어가다보니 코드 이해가 좀어려웠다ㅠㅠ  
이번엔 차례대로 코드 흐름을 따라가보자.

### Main Process &gt; main.js

```javascript
mainWindow.webContents.send('message-received', message); //렌더러로 내려보냄
```

### 브릿지 &gt; preload.js

IPC통신을 통해 'message-received'를 받은 것 까진 오케이.  
근데 onMessageReceived가 왜 밖에 선언돼있지…? callback이 onMessageReceived가 아닌건가?  
여기서 한참을 헤멨다…ㅠㅠ 일단은 이렇구나 하고 넘어가고 모듈의 chat.js에서 설명하겠다.

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('ChatInterface', {
  //sendMessage: (text) => ipcRenderer.send('send-message', text),

  onMessageReceived: (callback) => {
    ipcRenderer.on('message-received', (event, message) => {
      callback(data);
    });
  }
});
```

### 모듈 &gt; chat.js

메시지 이벤트를 받아서 실행하는 부분이다.

중요한 건, 여기선 브릿지에게 `콜백함수`를 전달한다는 거다.  
ChatInterface의 onMessageReceived을 통해 '너 이 이벤트 받으면 내 함수중에 이거 실행해줘!' 라고 등록한다.

즉 , 인터페이스의 함수는

> onMessageReceived = 콜백을 등록해두는 함수

라는 거다… 나는 이걸 실제로 이벤트를 발생시키는 함수라고 생각해서 약간의 혼동이 있었다.

```javascript
window.ChatInterface.onMessageReceived((data) => {displayMessage(data);});
```

생각보다… 복잡한데….?  
모바일 클라이언트와의 연동이 걱정되기 시작했다…

# 2\. **Mobile (Android)**: Native Android (Java)

모바일은 시간 상 나중에 정리예정…헷