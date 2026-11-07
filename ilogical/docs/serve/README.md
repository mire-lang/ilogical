# owl serve — Native WebSocket/HTTP Server

## Usage

```bash
owl serve [OPTIONS] <IP:PORT>
```

## Options

| Option | Description |
|--------|-------------|
| `--ws` | Enable WebSocket support |
| `--http` | Enable HTTP support |
| `--dir <DIR>` | Static file directory |
| `--cert <FILE>` | TLS certificate file |
| `--key <FILE>` | TLS key file |

## Examples

```bash
# HTTP server on port 8080
owl serve 0.0.0.0:8080

# WebSocket server on port 8080
owl serve 0.0.0.0:8080 --ws

# Both HTTP and WebSocket with static files
owl serve 0.0.0.0:8080 --http --ws --dir ./public

# HTTPS with TLS
owl serve 0.0.0.0:8443 --http --ws --cert cert.pem --key key.pem
```

## Behavior

1. Binds TCP listener on specified IP:PORT
2. Accepts connections in loop
3. Reads first request to detect protocol:
   - `Upgrade: websocket` → WebSocket handshake
   - Otherwise → HTTP request (if `--http`)
4. Serves static files from `--dir` for HTTP GET
5. Handles WebSocket frames for WS connections

## WebSocket Upgrade

Detects `Upgrade: websocket` header, performs handshake:
1. Reads `Sec-WebSocket-Key`
2. Computes `Sec-WebSocket-Accept`
3. Sends `101 Switching Protocols`
4. Enters WebSocket frame loop

## Static Files

When `--dir` specified, serves files relative to directory:
- `GET /file.txt` → `$DIR/file.txt`
- `GET /` → `$DIR/index.html` (if exists)
- 404 if not found

## TLS

When `--cert` and `--key` provided, uses TLS for both HTTP and WebSocket (wss://).

## Implementation

Entry point: `ilogical::serve::main(args :vec[str]) :i64`
Uses `kioto::net` for TCP, `kioto::net::ws` for WebSocket.
