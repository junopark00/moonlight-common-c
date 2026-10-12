# Moonlight Streaming Core Library

Moonlight-common-c contains the core GameStream client code shared between [Moonlight](https://moonlight-stream.org) clients, including [Moonlight PC](https://github.com/moonlight-stream/moonlight-qt), [Moonlight Android](https://github.com/moonlight-stream/moonlight-android), [Moonlight iOS](https://github.com/moonlight-stream/moonlight-ios), and [Moonlight Chrome](https://github.com/moonlight-stream/moonlight-chrome).

If you are implementing your own Moonlight game streaming client that can use a C library, you probably want the code here.

## Note to Developers

Moonlight-common-c requires the _specific_ version of ENet that is bundled as a submodule. This version has changes required for IPv6 compatibility and retransmission reliability, among other things. These are breaking API/ABI changes which make Moonlight-common-c incompatible with other versions of the ENet library. Attempting to runtime link to another libenet library will cause your client to crash when connecting to recent versions of GeForce Experience.

## Hermit extension: cursor messages

This fork (branch `hermit`) adds one optional connection listener callback, `cursorPacket`, for the
local cursor feature of [Hermit](https://github.com/junopark00/hermit) and
[Shell](https://github.com/junopark00/hermit-shell). A Shell host that the client asked for it
(over HTTPS, outside this library) sends two control stream packet types on the reliable, encrypted
control channel:

| Type | Name | Content |
|---|---|---|
| `0x3100` | `CURSOR_SHAPE` | the host cursor's image (BGRA, straight alpha) and hotspot |
| `0x3101` | `CURSOR_STATE` | a visibility change, or a switch to a shape sent earlier |

The library does not interpret these payloads. Both are dispatched through the existing asynchronous
callback queue, in arrival order and without batching, as
`cursorPacket(packetType, payload, length)`; `payload` is the data after the control packet header
and is valid only during the callback. Clients that leave `cursorPacket` NULL drop the messages (as
stock moonlight-common-c drops every unknown type), and hosts without the extension never send
them, so nothing changes for stock clients and hosts. The type numbers do not overlap the Sunshine
(`0x55xx`) or Apollo (`0x30xx`) extensions. The payload layouts are documented with the Shell host
(`docs/` in the Shell repository).

Because cursor shapes (up to five chunks each, 20 per second) and cursor states (60 per second) can
arrive in bursts, the asynchronous callback queue holds 256 entries in this fork instead of 30, and a
full queue is logged once per connection (the first dropped packet type) instead of on every drop.
