# Battery drain on iPhone: what was wrong and what changed

A fork of [kulikov0/whitelist-bypass](https://github.com/kulikov0/whitelist-bypass) (MIT). The bypass itself was designed and written by [@kulikov0](https://github.com/kulikov0); this fork only touches power consumption in the iOS client.

Base commit `0f9f908`, app version 0.3.8.

## Two loops spun the CPU for nothing

**The VP8 writer.** Frame pacing is `sampleInterval = (1s / fps) / pacedBatch`, so the defaults `fps=24`, `batch=30` give **1.54ms** — the ticker fires 648 times a second. The log says it outright:

```
vp8tunnel: writer (re)started fps=24 batch=30 pacedBatch=27 sampleInterval=1.543209ms keepaliveEvery=70
```

With nothing to send the loop still woke on every tick, only to bump a counter:

```go
default:
    idleTicks++
    if idleTicks < keepaliveEvery {
        continue
    }
```

A real frame goes out once every 70 ticks, about nine times a second. The other **639 wakeups per second did nothing** except keep the core out of deep sleep. With `dualTrack` there are two such writers.

**The KCP update loop.** With Reliable KCP enabled, `updateLoop` called `Update()` on a fixed 10ms ticker — **100 times a second regardless of traffic**. While idle there is nothing to retransmit: `WaitSnd()` is zero across every session.

Both loops now run at two cadences. While data flows the behaviour is unchanged and the traffic shape is identical. Once idle the ticker stops and the goroutine blocks; sends and receives wake it immediately, so the first packet after silence is not delayed.

## Measurements

Two things matter separately: CPU time, and how many packets actually go on the wire. On a phone the second one dominates — every frame is a packet that keeps the radio out of sleep.

`relay/tunnel/idle_rate_test.go`, single track, 30s of pure idle:

| | frames/s | CPU (share of one core) |
|---|---|---|
| original | 7.9 | 2.155 % |
| patched, same keepalive | 7.7 | 0.154 % |
| patched, keepalive 3–8s | **0.2** | **0.007 %** |

Reproduce it on either tree:

```sh
cd relay
go test ./tunnel/ -run TestIdleEmissionRate -v
KEEPALIVE_MS=3000,8000 go test ./tunnel/ -run TestIdleEmissionRate -v
```

The middle row is the honest limit of the loop fix on its own: CPU drops 14x, **but the packet rate barely moves**. Idle traffic is driven by the keepalive period, not by the ticker. That is why the keepalive knob matters more for battery than the loop rewrite does — together they take idle emissions from ~8 packets per second down to one every five seconds.

CPU-only comparison including the KCP loop, from `relay/benchidle`:

| | before | after |
|---|---|---|
| vp8 only | 2.47 % | 0.14 % |
| vp8 + Reliable KCP | 2.95 % | 0.15 % |

These are wire-level and CPU-level numbers. **Battery life in hours was not measured** — that needs the device, not a build machine.

## What is left untouched, and why

Three things still drive drain and none of them are fixable here:

1. **The silent audio session.** The app loops a silent buffer so iOS does not suspend it (`UIBackgroundModes: audio`), which keeps the audio subsystem alive around the clock. The buffer was generated at 44100Hz and is now 8000Hz — 5.5x fewer samples — but the session itself stays open.
2. **Two processes.** Traffic passes through this proxy app and a second client app on top of it, so every packet is copied between them.
3. **The LiveKit websocket ping** every 5 seconds. The interval is dictated by the server in its join response; shortening or dropping it gets the connection closed.

Points 1 and 2 both go away with `NEPacketTunnelProvider`: a network extension gets legitimate background execution and can route traffic itself. It requires a paid Apple Developer account — the network extension entitlement cannot be signed with a free Apple ID, and sideloading through AltStore does not change that.

## The rest of the changes

**Logging burned the main thread.** `onLog` is called from Go for every line, and an active tunnel produces hundreds per minute. Each one printed through `print()` in release builds too, hopped to the main thread, and mutated a `@Published` array — so SwiftUI recomputed the view even with the log collapsed and nobody watching. `showLogs` also defaulted to on.

**The keepalive period was nailed shut.** `SetKeepaliveShape` existed but was never called from anywhere, so the period stayed at 60–200ms forever. It is now carried from the app into the tunnel and exposed in settings; the default is 3–8s.

**The video track was always published.** `onLKReady` created a VP8 track and sent `AddTrack` unconditionally, and `startTunnel` started the writer unconditionally — the mode branch came only afterwards. In DC mode that entire path runs for nothing, since the payload travels over the data channel. `publishDataOnly` brings the publisher up with a data channel alone. **Off by default**: whether WB Stream accepts a participant without video can only be established against a live room.

**Minor.** mDNS candidate gathering is off — the peer is always on the internet, so `.local` addresses are dead weight.

## Branches

- [`idle-wakeups`](https://github.com/Jynk0-btw/whitelist-bypass/tree/idle-wakeups) — both idle loops plus the measurement test
- [`tunnel-options`](https://github.com/Jynk0-btw/whitelist-bypass/tree/tunnel-options) — tunable keepalive, DC without a video track, mDNS toggle
- [`ios-battery`](https://github.com/Jynk0-btw/whitelist-bypass/tree/ios-battery) — the app changes
- `battery-all` — everything together; the `.ipa` is built from here
