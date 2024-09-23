---
title: "WebRTC 개요"
seoTitle: "WebRTC 소개"
seoDescription: "WebRTC 개요: 미디어 캡처, 피어 연결 설정, 시그널링, ICE 후보 교환을 통한 실시간 통신 구성 방법"
datePublished: Sun Sep 22 2024 14:51:36 GMT+0000 (Coordinated Universal Time)
cuid: cm1dp5fjp000w08lcab9ya3g0
slug: webrtc
tags: webrtc

---

# **1\. Fetching**

---

```jsx
// getUserMedia()
const localStream = await getUserMedia({vide: true, audio : true});

// RTCPeerConnection 생성
const peerConnection = new RTCPeerConnection(iceConfig);

// addTrack() 미디어 스트림 추가
localStream.getTracks().forEach(track => {
    peerConnection.addTrack(track, localStream);
});
```

## 1) getUserMedia()

> 미디어 캡처 및 스트림을 획득하기 위한 WebRTC 지원 함수

사용자의 카메라와 마이크같은 미디어 입력장치에 접근하여, 비디오와 오디오 스트림을 캡처할 수 있게

* 미디어 스트림을 반환
    
* 반환된 스트림은 비동기적으로 처리
    
* 프로미스를 통해 접근 가능
    

`.then()`이나 `async/await`를 사용하여 스트림을 받아와 동기적으로 처리

## 2) RTCPeerConnection 객체 생성

> Peer 간의 연결을 설정하고 미디어 스트림을 주고받기 위한 객체 준비

WebRTC에서 피어 간의 연결을 설정하기 위해 **RTCPeerConnection** 객체를 생성

`iceConfig`는 ICE (Interactive Connectivity Establishment) 서버 설정을 포함하는 객체

## 3) addTrack()

> 미디어 스트림을 RTCPeerConnection에 추가

사용자의 오디오나 비디오 데이터(스트림)를 WebRTC 연결을 통해 다른 피어(상대방 브라우저)로 전달할 수 있도록 `RTCPeerConnection` 객체에 스트림을 연결하는 것

* **미디어 스트림** : 카메라, 마이크 등의 입력 장치에서 가져오는 영상이나 음성 데이터
    
* 이 스트림을 `RTCPeerConnection`에 추가함으로써 **실시간으로 다른 사람과 공유할 준비를 마친다**는 것이 핵심
    

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><strong>스트림 ?</strong> 자신이 상대방에게 보내고자 하는 실제 미디어 데이터</div>
</div>

---

## 여기까지 한 일 ?

상대방에게 캡처한 이 스트림 보낼거고 보낼 준비가 됐어!

BUT, 연결을 위한 정보(SDP, ICE 후보)는 아직 없는 상태

# **2\. Signaling**

---

```jsx
peerConnection.createOffer().then((offer) => {
  // 오퍼 생성 후 로컬 SDP로 설정
  return peerConnection.setLocalDescription(offer);
}).then(() => {
  // 오퍼를 상대방에게 전송 (예: signaling 서버를 통해)
  sendOfferToPeer(peerConnection.localDescription);
});
```

피어 간 연결을 설정하기 위해 필요한 초기 정보(e.g. SDP, ICE 후보)를 **안전하고 효과적으로 교환**하기 위해서 **시그널링 서버와의 통신**을 진행

WebRTC는 미디어 데이터를 피어 간에 직접 주고받지만, **연결 초기 단계**에선 서로를 찾거나 직접 연결할 방법이 없기 때문에 **중간 매개체인 시그널링 서버가 필요**

자신의 연결 정보를 전달하기 위해 offer를 생성

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><strong>Offer ? </strong>SDP(Session Description Protocol) 포맷으로 세션 정보를 담고 있음 지원하는 미디어 형식, 코덱 정보, 네트워크 설정…등의 미디어 정보</div>
</div>

## 1) createOffer()

> PEER A(보내는 쪽)가 어떤 미디어를 지원하는지 지원 코덱은 어떤 건지 등등을 탐색

* 자신의 지원 코덱정보를 알게 되는 순간
    

”이 스트림을 보내려고 준비 중이고, 나는 이런 코덱과 미지어 형식을 지원하고 있으니까 너 참고해!"라는 정보를 담은 **SDP**를 상대방에게 제안하기 위해 보낼 데이터를 준비하는 과정

## 2) setLocalDescription()

> 로컬 SDP(Session Description Protocol)를 설정

> **오퍼(Offer)** 또는 \*\*응답(Answer)\*\*을 생성한 후, 자기 자신의 피어에게 설정하기 위한 지원 함수

* **자신의 SDP 정보를 로컬에 설정**해, 브라우저가 그 정보를 사용하도록!
    
* **오퍼나 응답을 전송하기 전 준비 과정**으로, SDP를 내부적으로 확정
    
* SDP가 설정되면, 브라우저는 **ICE 후보 수집 과정**에 필요한 조건을 충족하게 됨
    

”이게 내 미디어 및 네트워크 설정이다”라고 **자기 자신에게 확정하는 것**

상대의 정보를 받기 전에 응답을 처리할 방법을 자신이 알고 있어야하기 때문에, 그 전에 자신의 SDP가 설정돼있어야 함

받는 방법을 모르는데 어떻게 받아?  
따라서 피어에 SDP가 설정돼있지 않으면 의미없는 통신이 된다!

### 굳이 createoffer와 분리한 이유?

상대 PEER가 다른 제안을 할 수도 있고, 미디어 설정을 변경할 필요가 생길 수도 있음  
이러한 다양한 상황에서 유연하게 대처하기 위해 로컬 SDP를 바로 확정하지 않는 것

## 3) sendOffer : 소켓 통신 전달

```jsx
sendOfferToPeer(peerConnection.localDescription);
//offer를 보내는 것이 아니라 확정된 정보를 보내야 한다.
```

* `createOffer()`로 생성된 SDP는 **초안**에 해당하고, `setLocalDescription()`을 통해 브라우저가 이를 실제로 **사용할 준비가 된 상태로 확정**
    
* 이 **확정된 정보**가 `peerConnection.localDescription`에 저장되고, 이 **최종 확정된 SDP**를 상대방에게 보내는 것이 더 정확
    

## 4) setRemoteDescription()

> 원격 SDP(Session Description Protocol)를 설정

* `setLocalDescription()`: 내가 **생성한 SDP**를 **내 브라우저**에 설정  
    브라우저는 내가 전송할 미디어 설정(코덱, 해상도 등)을 확정
    
* `setRemoteDescription()`: 상대방이 **생성한 SDP**를 **내 브라우저**에 설정  
    브라우저는 상대방의 설정을 받아들여 그에 맞춰 미디어 스트림을 처리
    

## e.g. P2P통신

PEER A와 PEER B가 이미 시그널링 서버에 접속한 상태에서 생성한 Offer를 서버를 통해 전달

P2P방식의 경우, 상대방에게 해당 데이터를 보내고 싶다고 SDP를 전달

오퍼를 받은 PEER B는 PEER A로부터 수신한 정보를 통해 setRemoteDescription()을 진행하고, PEER A가 했던 것 처럼 동일하게 `peerConnection`을 생성하고 자신의 정보를 설정(단, 여기선 Offer대신 createAnswer()로 응답을 생성)

응답을 받은 PEER A역시 setRemoteDescription()을 통해 로컬에 설정

---

## 여기까지 한 일 ?

오퍼(Offer)와 응답(Answer)을 교환하여 메시지 프로토콜 설정 완료

BUT, 여기까지 와도 미디어 데이터 자체를 전송하는 방법은 아직 모른다!

# 3\. ICE

---

```jsx
peerConnection.onicecandidate = (event) => { // WebRTC 지원 함수
  if (event.candidate) {
    // ICE 후보가 생성되면 시그널링 서버를 통해 피어 B에게 전달
    signalingServer.send(JSON.stringify({ candidate: event.candidate }));
  }
};
```

> Interactive Connectivity Establishment(인터넷 연결 생성)

> ICE candidate : webRTC에 필요한 프로토콜(ICE 후보)

* **SDP**는 **세션 설정**에 대한 정보(어떤 방식으로 주고받을래?)
    
* **ICE**는 **네트워크 연결**을 설정하는 과정(어디서 만날래?)
    

## **ICE 후보(ICE candidate)**

> **ICE 후보(ICE candidate)** 가 생성됨  
> \= **피어 간의 연결을 위한 네트워크 경로(IP 주소 및 포트 번호)**가 찾아짐

**ICE 후보**는 **피어 간의 통신을 시도할 수 있는 다양한 네트워크 경로**

두 피어는 각자의 **ICE 후보**를 서로 교환한 후, **서로 연결할 수 있는 최적의 경로를 선택함**

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">이미 스트리밍을 시작했더라도 모든 목록이 전송완료될 때 까지 계속 ICE 정보를 보냄</div>
</div>

## **Candidate 종류**

* **로컬 후보(Local Candidate)**: 로컬 네트워크 내에서 통신할 수 있는 IP와 포트.
    
* **STUN 후보(Server Reflexive Candidate)**: NAT 뒤에 있는 피어가 공인 IP를 얻기 위해 STUN 서버를 통해 획득한 경로.
    
* **TURN 후보(Relayed Candidate)**: 피어 간 직접 통신이 불가능할 때 TURN 서버를 통한 중계 경로.
    

## ICE에서 STUN과 TURN 서버가 필요한 이유

PEER A에서 PEER B까지 단순하게 연결하는 것으로는 작동하지 않을 수 있다는 사실을 명심하자

네트워크 환경은 가변적이고 복잡하기 때문에,  
NAT(Network Address Translation)와 방화벽이 있는 환경에서는 직접적인 피어 간 연결이 어렵다!

### **STUN 서버**

* NAT 뒤에 있는 피어가 공인 IP 주소를 얻기 위해 사용
    
* STUN 서버는 피어가 자신의 공인 IP 주소와 포트를 확인할 수 있게 도와줌
    

### **TURN 서버**

* 피어 간 직접 통신이 불가능할 때 사용
    
* 중계 서버 역할을 하여, 피어 간의 데이터를 중계함
    
* 특히 방화벽이나 대칭형 NAT 환경에서 유용
    

## Check Point

> Q. 양방향 통신을 할 때 로컬 유저(PEER A)와 원격 유저(PEER B)인 경우와 로컬 유저(PEER B)와 원격 유저(PEER A)인 경우 통신 candidiate가 다를 수도 있나?
> 
> A. 그렇다.

---

# 미디어 전송을 위한 정보 교환 완료

> ICE 후보 교환까지 완료될 경우 미디어 데이터 전송이 가능해짐

ICE 후보 교환을 통해 피어 간의 최적의 네트워크 경로가 설정되면,  
WebRTC 연결이 확립되고,  
이를 통해 오디오, 비디오 등의 미디어 데이터를 실시간으로 주고받을 수 있게 된다!