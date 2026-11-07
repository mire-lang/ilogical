# kioto::net::ws — WebSocket Implementation

## Client

```mire
pub fn connect: (url :&str) :WebSocket
```

Connects to `ws://` or `wss://` URL, performs handshake automatically.

## Server

```mire
pub fn serve: (port :u16) :Server
pub fn accept: (server :&Server) :WebSocket
```

Creates TCP listener, accepts WebSocket connections with handshake.

## WebSocket Struct

```mire
pub struct WebSocket {
    socket :net::Socket
    is_client :bool
    mask_outgoing :bool
}
```

## Frame Operations

```mire
pub fn send_text: (ws :&WebSocket, text :&str) :i64
pub fn send_binary: (ws :&WebSocket, data :&ptr, len :i64) :i64
pub fn send_ping: (ws :&WebSocket, data :&ptr, len :i64) :i64
pub fn send_pong: (ws :&WebSocket, data :&ptr, len :i64) :i64
pub fn send_close: (ws :&WebSocket, code :i64, reason :&str) :i64
pub fn recv_frame: (ws :&WebSocket) :Frame
```

## Frame Structure

```mire
pub struct Frame {
    fin :bool
    rsv1 :bool
    rsv2 :bool
    rsv3 :bool
    opcode :i64
    masked :bool
    payload_len :i64
    masking_key :i64
    payload :ptr
    payload_len :i64
}
```

## Opcodes

| Constant | Value | Description |
|----------|-------|-------------|
| `WS_OP_CONTINUATION` | 0x0 | Continuation frame |
| `WS_OP_TEXT` | 0x1 | Text frame |
| `WS_OP_BINARY` | 0x2 | Binary frame |
| `WS_OP_CLOSE` | 0x8 | Close frame |
| `WS_OP_PING` | 0x9 | Ping frame |
| `WS_OP_PONG` | 0xA | Pong frame |

## Close Codes

Per RFC 6455: 1000 (normal), 1001 (going away), 1002 (protocol error), 1003 (unsupported data), 1005 (no status), 1006 (abnormal), 1007 (invalid payload), 1008 (policy violation), 1009 (too big), 1010 (extension required), 1011 (unexpected).

## Handshake

Client generates `Sec-WebSocket-Key` (16 random bytes, base64). Server responds with `Sec-WebSocket-Accept` = base64(SHA1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11")).

## Implementation Status

Frame encoding/decoding delegates to PAL (`rt_ws_encode_frame`, `rt_ws_decode_frame`). Runtime implementation needed.
