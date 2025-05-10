---
title: "Ui 공통 모듈...을 만들게 됐어요"
datePublished: Sat May 10 2025 14:53:25 GMT+0000 (Coordinated Universal Time)
cuid: cmaichon3000109jv10401bxr
slug: chat-ui-module

---

우당탕탕 프론트 일기 1탄…

현재 서비스중인 프로그램엔 모바일 클라이언트와 윈도우 클라이언트가 각각 존재한다. 근데 두 클라이언트가 별도로 개발돼있어서 디자인 작업을 두 번 해야 하는 번거로움이 있어 이번에 일부 기능을 합치기로 했다. 채팅 기능인데, 덕분에 기존 채팅관련 소스를 파악하는 중이라 재밌고 신기해하는 중. 채팅 기능을 첨 접해봐가지구…헷

모듈 내부에 큰 기능이 있는 건 아니고 정말 UI만 지원하는 거라 역할을 분리해서 생각하는 게 중요하다. (서버 통신이나 데이터 저장은 각 클라이언트의 역할이고, 그나마 좀 중요한 기능이라고 하면 UI 상의 숫자계산정도? 이건 각 클라이언트에서 두 번 계산식 작성하느니 모듈에서 작업하는 게 맞을 것 같다.)

그래서 각 클라이언트에서 공통으로 호출할 인터페이스를 작성해야한다. 그러기 위해선 각 클라이언트에서 공통 모듈에 어떻게 접근 가능한지, 내가 두 클라이언트를 지원하려면 어떻게 인터페이스를 구성해야할 지 알아두는 게 도움이 될 거다. 어떻게 연결하는 지를 알아야 다리를 만들어 줄 수 있으니깐…!

## 🍀 지원 클라이언트

* **Desktop (Windows)**: Electron (Chromium 기반 데스크탑 앱)
    
* **Mobile (Android)**: Native Android (Java)
    

## 0\. **모듈 구조**

* chat.html
    
* chat.js
    

모듈에는 두 파일이 있다고 일단 가정하자.

# 1\. **Desktop (Windows)** : Electron (Chromium 기반 데스크탑 앱)

참고로 PC client는 어차피 내가 해야한다. 왜냐면 이 쪽 개발도 내가 담당하고있기 때문이다. (나는 나와 통신한다…)

Electron 환경에서 가장 유의해야할 점은 보안이다. 기본 보안 설정에서는 HTML파일만 로드해선 내부에 선언된 JS 파일을 실행시킬 수 없다. 따라서 필요한 함수는 미리 **preload 스크립트**를 통해 명시적으로 등록해줘야한다.

## 1) 모듈 → Electron 통신

### preload.js

> **Electron에서 웹 페이지(렌더러)와 Node.js 기능(메인 프로세스/IPC) 사이를 안전하게 연결해주는 스크립트**  
> **앱이 로드되기 전에 실행되며, 필요한 기능만 노출해주는 브릿지 역할**

ChatInterface 라는 이름의 전역 객체를 window에 등록하는 작업이고, 이 ChatInterface 인터페이스를 통해 chat.js에서 이벤트를 보낼 수 있는 거다. 야호!

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('ChatInterface', {
  sendMessage: (text) => ipcRenderer.send('send-message', text)
});
```

### chat.js

```javascript
window.ChatInterface.sendMessage(text);
```

## 2) Electron → 모듈 통신

### preload.js

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('ChatInterface', {
  sendMessage: (text) => ipcRenderer.send('send-message', text),

  onMessageReceived: (callback) => {
    ipcRenderer.on('message-received', (event, data) => {
      callback(data);
    });
  }
});
```

### chat.js

```javascript
window.ChatInterface.onMessageReceived((data) => {
  displayMessage(data);
});
```

### Electron 메인 프로세스

```javascript
ipcMain.on('send-message', (event, text) => {
  console.log('메시지 수신:', text);

  // 예: 받은 메시지를 다시 렌더러에 내려보냄
  const win = BrowserWindow.getFocusedWindow();
  win.webContents.send('message-received', {
    from: 'server',
    content: '서버에서 전달: ' + text,
  });
});
```

모바일은 시간 상 나중에 정리예정…헷