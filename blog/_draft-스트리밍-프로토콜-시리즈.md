# Streaming Protocols — 시리즈 계획

## 시리즈 목표

"데이터를 실시간으로 흘려보내는" 다양한 방식을 비교하고,
각 프로토콜이 어떤 문제를 풀기 위해 등장했는지를 이해한다.

---

## 편 구성 (안)

### TCP 위의 스트리밍: Long Polling, SSE, WebSocket

- TCP Keep-Alive: 연결을 어떻게 유지하는지
- Long Polling 동작 원리와 한계
- SSE: HTTP 위에서 단방향 이벤트 스트림
- WebSocket: HTTP 업그레이드로 양방향 전환
- 세 방식 비교 — 언제 무엇을 쓸까

### HTTP 적응형 스트리밍: HLS와 DASH

- .ts 파일의 정체: MPEG-2 Transport Stream 세그먼트
- HLS 동작 원리: 매니페스트(.m3u8) → 세그먼트 요청 → 재생
- DASH와의 비교
- ABR(Adaptive Bitrate): 네트워크 상황에 따라 화질 전환
- CDN과의 결합

### UDP 기반 실시간 전송: RTP / RTCP / RTSP

- UDP를 왜 실시간 미디어에 쓰는가
- RTP: 미디어 데이터 전송
- RTCP: 품질 피드백 (패킷 손실, 지터)
- RTSP: 재생 제어 (VoD, CCTV)
- SRT: RTMP를 대체하는 UDP 기반 ingest 프로토콜
- QUIC과의 관계 → HTTP/3 편 참조

### P2P 스트리밍: WebRTC

- 브라우저끼리 직접 연결: ICE, STUN, TURN, SDP
- 내부적으로 UDP(DTLS + SRTP) 사용
- NAT 뒤에서 연결하는 방법
- BitTorrent 계열 P2P와의 차이 (간략)

---

## 각 편에서 공통으로 다룰 것

- 해당 프로토콜이 등장한 배경 (어떤 문제를 풀려 했나)
- 핵심 동작 원리 (너무 깊지 않게, 이해에 필요한 수준)
- 실제 사용 사례
- 다른 프로토콜과의 비교 / 언제 쓰면 좋은가

---

## 시리즈 흐름 메모

```
TCP Keep-Alive
  └─ HTTP Streaming (chunked)    ← 1편
  └─ SSE (단방향)                ← 1편
  └─ WebSocket (양방향)          ← 1편

HTTP 위 적응형 스트리밍
  └─ HLS (.ts 세그먼트)          ← 2편
  └─ DASH                        ← 2편

UDP
  └─ RTP/RTCP/RTSP               ← 3편
  └─ QUIC (신뢰성 추가)          ← HTTP/3 편 참조

P2P
  └─ WebRTC (UDP 기반)           ← 4편
  └─ BitTorrent (언급)           ← 4편
```

---

## 메모 / 미결 사항

- HTTP/2 Server Push는 넣을지? (SSE와 겹치는 느낌, 일단 보류)
- QUIC은 3편에서 짧게 언급하고 HTTP/3 편 링크로 처리
- 4편 이후 "언제 어느 프로토콜을 쓸까" 비교 정리 편을 추가할지 고려
