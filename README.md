# stratus

A Gleam WebSocket client (and possibly more).

[![Package Version](https://img.shields.io/hexpm/v/stratus)](https://hex.pm/packages/stratus)
[![Hex Docs](https://img.shields.io/badge/hex-docs-ffaff3)](https://hexdocs.pm/stratus/)

```sh
gleam add stratus
```

## Features
- [x] Sending and receiving Text and Binary frames
- [x] Custom user messages
- [ ] Sending and receiving continuation frames
- [ ] Per-message deflate
- [ ] Graceful shutdown

## Example

```gleam
import birl
import gleam/erlang/process
import gleam/http/request
import gleam/io
import gleam/option.{None}
import gleam/otp/actor
import repeatedly
import stratus

pub type Msg {
  Close
  SendText(String)
}

pub fn main() {
  let assert Ok(req) = request.to("http://localhost:3000/ws")
  let builder =
    stratus.websocket(
      request: req,
      init: fn() { #(Nil, None) },
      loop: fn(msg, state, conn) {
        case msg {
          stratus.Text(_msg) -> {
            let assert Ok(_resp) =
              stratus.send_text_message(conn, "hello, world!")
            actor.continue(state)
          }
          stratus.User(SendText(msg)) -> {
            let assert Ok(_resp) = stratus.send_text_message(conn, msg)
            actor.continue(state)
          }
          stratus.Binary(_msg) -> actor.continue(state)
          stratus.User(Close) -> {
            let assert Ok(_) = stratus.close(conn)
            actor.Stop(process.Normal)
          }
        }
      },
    )
    |> stratus.on_close(fn(_state) { io.println("oh noooo") })

  let assert Ok(subj) = stratus.initialize(builder)

  let timer =
    repeatedly.call(1000, Nil, fn(_state, _count_) {
      let now =
        birl.now()
        |> birl.to_iso8601
      stratus.send_message(subj, SendText(now))
    })

  process.sleep(6000)

  stratus.send_message(subj, Close)

  repeatedly.stop(timer)
}
```

Further documentation can be found at <https://hexdocs.pm/stratus>.
