---
title: 'TCP 위의 스트리밍: Long Polling, SSE, WebSocket'
pubDate: 'May 19 2026'
tags: ['Network']
series: 'streaming-protocol'
seriesOrder: 1
heroImage: '../assets/2026-05-19-Streaming-Protocols/1.png'
description: 'Steaming Procotols 시리즈 1편 - TCP 위에서 실시간 데이터를 전달하는 세 가지 방식 비교'
---

## 개요

이 글은 TCP 연결 위에서 실시간 데이터를 전달하는 세 가지 방식을 다룬다.
 
 - 응답을 의도적으로 지연시켜 서버 푸시를 흉내 내는 Long Polling
 - HTTP 연결을 열어두고 서버가 단방향으로 이벤트를 보내는 SSE
 - 그리고 HTTP 연결을 양방향 채널로 전환하는 WebSocket

세 방식이 어떤 문제를 풀기 위해 등장했는지, 그리고 어떤 상황에서 무엇을 선택하면 좋은지를 비교한다.

## TCP Keep-Alive — 연결을 살려두는 방법

TCP 연결은 기본적으로 데이터를 주고받지 않으면 OS나 중간 네트워크 장비(방화벽, NAT)가 유휴 연결을 끊어버릴 수 있다. Keep-Alive는 이를 막기 위해 일정 시간 데이터가 없을 때 연결 유지 확인용 패킷(프로브)을 주기적으로 보내는 메커니즘이다.

OS 레벨에서 소켓 옵션 `SO_KEEPALIVE`으로 설정하며, 세 가지 파라미터로 동작을 제어한다.

- `tcp_keepalive_time`: 마지막 데이터 교환 후 첫 프로브를 보내기까지 대기 시간 (기본 2시간)
- `tcp_keepalive_intvl`: 프로브 전송 간격
- `tcp_keepalive_probes`: 응답 없을 때 재시도 횟수, 초과 시 연결 종료

한 가지 주의할 점은 HTTP `Connection: keep-alive` 헤더와 혼동하기 쉽다는 것이다. HTTP Keep-Alive는 애플리케이션 레벨에서 하나의 TCP 연결로 여러 HTTP 요청을 처리하기 위한 것이고, TCP Keep-Alive는 그보다 아래 레이어인 OS 레벨에서 연결의 생존을 확인하는 것이다. 둘은 별개로 동작한다.

이 글에서 다루는 Long Polling, SSE, WebSocket은 모두 TCP 연결을 일정 시간 이상 열어두는 방식이기 때문에 Keep-Alive 설정이 그 기반이 된다.

## Long Polling

개발 경험이 짧다보니, 전설(?)로만 듣던 방식이기도 하다. HTTP는 기본적으로 클라이언트가 요청을 보내야만 서버가 응답할 수 있다. 서버가 먼저 클라이언트에게 데이터를 밀어넣는 구조가 아니기 때문에, 실시간 데이터를 전달하려면 다른 방법이 필요했다.

가장 단순한 해결책은 폴링이다. 클라이언트가 일정 주기로 계속 요청을 보내 새 데이터가 있는지 확인하는 방식이다. 하지만 데이터가 없어도 요청이 계속 발생하므로 낭비가 크다. Long Polling은 이 문제를 줄이기 위해 등장한 변형이다.

### 동작 원리

클라이언트가 요청을 보내면, 서버는 즉시 응답하지 않고 새 데이터가 생길 때까지 응답을 보류한다. 데이터가 준비되면 그때 응답을 보내고, 클라이언트는 응답을 받자마자 곧바로 다음 요청을 보낸다. 이 과정을 반복하면 서버가 데이터를 준비되는 즉시 전달하는 것처럼 동작한다.

```
Client          Server
  |──── GET /poll ────▶|
  |                    | (데이터 없음, 대기)
  |                    | (데이터 생성)
  |◀─── 200 OK ────────|
  |──── GET /poll ────▶|  ← 즉시 재요청
  |                    |
```

### 한계

매 응답마다 HTTP 요청이 새로 시작되므로, 헤더를 포함한 연결 수립 비용이 반복된다. 데이터가 자주 발생하는 상황에서는 일반 폴링과 비용 차이가 거의 없어진다. 또한 서버 입장에서는 다수의 연결을 장시간 열어두기 때문에 리소스 부담이 크다.

SSE와 WebSocket이 등장하면서 Long Polling은 레거시 브라우저 지원이나 단순한 환경에서만 남게 되었다.

## SSE (Server-Sent Events)

SSE는 하나의 HTTP 연결을 열어두고 서버가 클라이언트로 이벤트를 지속적으로 밀어넣는 방식이다. Long Polling처럼 요청을 반복하지 않고, 연결 하나를 유지하면서 데이터가 생길 때마다 서버가 전송한다.

### 동작 원리

클라이언트가 `EventSource` API로 연결을 열면, 서버는 `Content-Type: text/event-stream`으로 응답하고 연결을 닫지 않는다. 이후 서버는 원하는 시점에 이벤트를 텍스트 형식으로 스트리밍한다.

```
Client                    Server
  |── GET /events ───────▶|
  |◀── 200 text/event-stream ──|
  |◀── data: hello\n\n ────|
  |◀── data: world\n\n ────|
  |         (연결 유지)
```

단방향(서버 → 클라이언트)이며, 브라우저가 `EventSource`를 기본 지원한다.

### 이벤트 형식

SSE는 `\n\n`으로 구분되는 텍스트 블록 형식을 사용한다.

```
data: 단순 메시지

event: user-joined
data: {"name": "Alice"}

id: 42
data: ID가 있는 이벤트

retry: 3000
data: 재연결 대기시간 3초로 설정
```

- `data`: 전송할 내용 (여러 줄이면 `data:`를 반복)
- `event`: 커스텀 이벤트 타입, 생략 시 `message`
- `id`: 이벤트 식별자, 재연결 시 활용
- `retry`: 연결 끊겼을 때 재시도 대기시간 (밀리초)

### 재연결과 Last-Event-ID

SSE의 큰 장점 중 하나는 브라우저가 연결 끊김을 감지하면 자동으로 재연결을 시도한다는 점이다. 재연결 시 마지막으로 수신한 이벤트의 `id`를 `Last-Event-ID` 헤더에 담아 서버에 보낸다. 서버는 이 값을 보고 누락된 이벤트를 다시 전송할 수 있다.

```
GET /events
Last-Event-ID: 42
```

## WebSocket

SSE가 단방향 스트리밍에 최적화되어 있다면, WebSocket은 클라이언트와 서버가 동시에 메시지를 주고받을 수 있는 양방향 채널이다.

### HTTP에서 WebSocket으로 업그레이드

WebSocket 연결은 HTTP 요청에서 시작된다. 클라이언트가 `Upgrade` 헤더를 포함한 요청을 보내고, 서버가 `101 Switching Protocols`로 응답하면 이후부터는 HTTP가 아닌 WebSocket 프로토콜로 통신한다.

```
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

핸드셰이크가 완료되면 TCP 연결은 그대로 유지되지만 HTTP는 사용하지 않는다.

### 프레임 구조와 양방향 통신

WebSocket은 데이터를 프레임 단위로 전송한다. 각 프레임은 헤더와 페이로드로 구성되며, 헤더에는 메시지 종류를 나타내는 opcode가 포함된다.

주요 opcode:
- `0x1`: 텍스트 프레임
- `0x2`: 바이너리 프레임
- `0x8`: 연결 종료
- `0x9` / `0xA`: Ping / Pong (연결 유지 확인)

클라이언트에서 서버로 보내는 프레임은 반드시 마스킹 처리된다. 이는 프록시 서버의 캐시 오염 공격을 막기 위한 규칙이다.

## 비교 — 언제 무엇을 쓸까

| | Long Polling | SSE | WebSocket |
|---|---|---|---|
| 방향 | 단방향 (서버→클라이언트) | 단방향 (서버→클라이언트) | 양방향 |
| 프로토콜 | HTTP | HTTP | WS (HTTP 업그레이드) |
| 연결 재사용 | 매 응답마다 재연결 | 단일 연결 유지 | 단일 연결 유지 |
| 브라우저 지원 | 모든 환경 | 대부분 지원 | 대부분 지원 |
| 자동 재연결 | 클라이언트 구현 필요 | 브라우저 내장 | 클라이언트 구현 필요 |
| 적합한 사례 | 레거시 환경, 낮은 빈도 알림 | 단방향 알림, LLM 응답 스트리밍 | 채팅, 게임, 협업 툴 |

클라이언트가 데이터를 받기만 하면 충분한 경우라면 SSE가 단순하고 효율적이다. 클라이언트도 실시간으로 데이터를 보내야 한다면 WebSocket을 선택한다. Long Polling은 SSE나 WebSocket을 쓸 수 없는 레거시 환경에서의 차선책이다.
