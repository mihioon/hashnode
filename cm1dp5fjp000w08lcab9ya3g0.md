---
title: "WebRTC 개요"
datePublished: Sun Sep 22 2024 14:51:36 GMT+0000 (Coordinated Universal Time)
cuid: cm1dp5fjp000w08lcab9ya3g0
slug: webrtc

---

# **1\. Fetching**

---

# `getUserMedia()` 호출

미디어 캡처 및 스트림 획득, 스트림 반환 비동기 함수로 프로미스 반환, 실제 코드에서는 `.then()`이나 `async/await`를 사용하여 스트림을 받아옴

# `RTCPeerConnection` 생성

## `addTrack()` 를 통해 미디어 스트림을 `RTCPeerConnection`에 추가

"미디어 스트림을 RTCPeerConnection에 추가한다" 사용자의 오디오나 비디오 데이터(스트림)를 WebRTC 연결을 통해 다른 피어(상대방 브라우저)로 전달할 수 있도록 `RTCPeerConnection` 객체에 스트림을 연결한다

구체적으로, **미디어 스트림**은 카메라, 마이크 등의 입력 장치에서 가져오는 영상이나 음성 데이터를 의미하고, 이 스트림을 `RTCPeerConnection`에 추가함으로써 **실시간으로 다른 사람과 공유할 준비를 마친다**는 것이 핵심

> 스트림은 자신이 상대방에게 보내고자 하는 실제 미디어 데이터를 의미!

그니까 여기까지 상대방에게 이 스트림 보낼거고 보낼 준비가 됐어

```jsx
const localStream = await getUserMedia({vide: true, audio : true});
const peerConnection = new RTCPeerConnection(iceConfig);
localStream.getTracks().forEach(track => {
    peerConnection.addTrack(track, localStream);
});
```

* 그리고 이걸 기반으로 자신의 연결 정보를 전달하기 위해 offer를 생성해야함 (sdp포맷으로 세션 정보 담겨있음/지원하는 미디어 형식과 코덱정보등의 미디어 정보와 네트워크 설정)
    

# **2\. Signaling**

---

WebRTC에서 피어 간 **직접 연결(P2P)**을 설정하기 위해 필요한 초기 정보(예: SDP, ICE 후보)를 **안전하고 효과적으로 교환하기 위해서 서버통신**

WebRTC는 미디어 데이터를 피어 간에 직접 주고받지만, **연결 초기 단계**에서 피어들끼리는 서로를 찾거나 직접 연결할 방법이 없기 때문에 **중간 매개체인 시그널링 서버가 필요**

## `createOffer()` 하는 순간 코덱정보를 알게 됨

> 여기까지 PEER A가 어떤 미디어를 지원하는지 지원 코덱은 어떤 건지 등등을 탐색하는 것 (아직까진 자기 본인만 알고있음)

”나는 이런 코덱과 미디어 형식을 지원할 거야, 너도 참고해!"라는 정보를 담은 **SDP**를 상대방에게 제안하기 위해 보낼 데이터를 준비하는 것

```jsx
peerConnection.createOffer().then((offer) => {
  // 오퍼 생성 후 로컬 SDP로 설정
  return peerConnection.setLocalDescription(offer);
}).then(() => {
  // 오퍼를 상대방에게 전송 (예: signaling 서버를 통해)
  sendOfferToPeer(peerConnection.localDescription);
});

```

이 스트림을 보내려고 준비 중이야, 그리고 나는 이러이러한 코덱과 설정을 사용할 거니까 확인해줘

## `setLocalDescription()`

로컬 SDP(Session Description Protocol)를 설정

**오퍼(Offer)** 또는 **응답(Answer)**을 생성한 후, 자기 자신의 피어에게 설정하는 것

”이게 내 미디어 및 네트워크 설정이다"이라고 **자기 자신에게 확정하는 것…**

굳이 createoffer와 분리한 이유는 상대방이 다른 제안을 할 수도 있고, 미디어 설정을 변경할 필요가 생길 수도 있으니까 이러한 다양한 상황에서 유연하게 대처하기 위해 바로 확정하지 않는 것

상대의 정보를 받기 전에 응답을 처리할 방법을 자신이 알고 있어야 하기 때문에, 그 전에 이미 자신의 SDP가 설정돼있어야 함(받는 방법을 모르는데 어떻게 받아 그러니까 피어에 SDP가 설정돼있지 않으면 의미없는 통신이 됨)

* **자신의 SDP 정보를 로컬에 설정**해, 브라우저가 그 정보를 사용하도록
    
* **오퍼나 응답을 전송하기 전 준비 과정**으로, SDP를 내부적으로 확정
    
* **ICE 후보 수집 과정**에 필요한 조건을 충족
    

# `sendOffer`

socket 통신으로 전달

```jsx
sendOfferToPeer(peerConnection.localDescription);
//offer를 보내는 것이 아니라 확정된 정보를 보내야 한다.
```

* `createOffer()`로 생성된 SDP는 **초안**에 해당하고, `setLocalDescription()`을 통해 브라우저가 이를 실제로 **사용할 준비가 된 상태로 확정**
    
* 이 **확정된 정보**가 `peerConnection.localDescription`에 저장되고, 이 **최종 확정된 SDP**를 상대방에게 보내는 것이므로 더 정확
    

peer a와 peer b는 이미 시그널링 서버에 접속한 상태고, p2p방식의 겨우 사용자에게 해당 데이터를 보내고 싶다고 sdp를 전달하는 거임

그럼 그걸 받은 peer b는 똑같이 peerconnection을 생성하고 자기 정보 설정하고 근데 로컬피어랑 원격피어랑 차이가 있으니까

setRemoteDescription();는 원격 설정을 알고 그에 맞춰 연결 준비할 수 있도록 하는 거임

그리고 p2p통신에선 서로 둘다 해야하는거고

* `setLocalDescription()`: 내가 **생성한 SDP**를 **내 브라우저**에 설정합니다. 이로 인해 브라우저는 내가 전송할 미디어 설정(코덱, 해상도 등)을 확정
    
* `setRemoteDescription()`: 상대방이 **생성한 SDP**를 **내 브라우저**에 설정합니다. 이로 인해 브라우저는 상대방의 설정을 받아들여 그에 맞춰 미디어 스트림을 처리
    

---

여기까지 offer와 answer을 교환하여 메시지 프로토콜 설정 완료함!

---

하지만 여기까지 와도 미디어 데이터 자체를 전송하는 방법은 아직 모름……

해당 연결을 하기 위해 ICE를 사용

# 3\. ICE

---

> Interactive Connectivity Establishment(인터넷 연결 생성) ICE candidate : webRTC에 필요한 프로토콜(ICE 후보)

**SDP**는 **세션 설정**에 대한 정보(어떤 방식으로 주고받을래?) **ICE**는 **네트워크 연결**을 설정하는 과정(어디서 만날래?)

### ICE에서 STUN과 TURN 서버가 필요한 이유

Peer A에서 Peer B까지 단순하게 연결하는 것으로는 작동하지 않을 수 있다.

💡

여기서 교환할 때 하나씩 차례대로 보낸다는 것 유의(리스트로 보내는 게 아니야)

검색되는 순서대로 candidate를 보낸다.

이미 스트리밍을 시작했더라도 모든 목록이 전송완료될 때 까지 계속 보냄

두 peer가 호환되는 candidiate를 제안하면 미디어 통신 시작

Q.

그럼 양방향 통신을 할 때 로컬 유저Peer A와 원격 유저Peer B인 경우와 로컬 유저Peer B와 원격 유저Peer A인 경우 통신 candidiate가 다를 수도 있나?

A. 그렇다.

```jsx
peerConnection.onicecandidate = (event) => {
  if (event.candidate) {
    // ICE 후보가 생성되면 시그널링 서버를 통해 피어 B에게 전달
    signalingServer.send(JSON.stringify({ candidate: event.candidate }));
  }
};

```

WebRTC에서 **ICE 후보(ICE candidate)** 가 생성되었다는 것은 **피어 간의 연결을 위한 네트워크 경로(IP 주소 및 포트 번호)**가 찾아졌음을 의미

해당 포트를 찾으면 이벤트를 발행하는 기능은 WebRTC에서 제공하는 내장 기능

두 피어는 각자의 **ICE 후보**를 서로 교환한 후, **서로 연결할 수 있는 최적의 경로를 선택**

**ICE 후보**는 **피어 간의 통신을 시도할 수 있는 다양한 네트워크 경로**

**IP 주소와 포트 정보**가 포함

* **로컬 후보(Local Candidate)**: 로컬 네트워크 내에서 통신할 수 있는 IP와 포트.
    
* **STUN 후보(Server Reflexive Candidate)**: NAT 뒤에 있는 피어가 공인 IP를 얻기 위해 STUN 서버를 통해 획득한 경로.
    
* **TURN 후보(Relayed Candidate)**: 피어 간 직접 통신이 불가능할 때 TURN 서버를 통한 중계 경로.