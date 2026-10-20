# Multiplayer Game Lobby API: Fan-Out Semantics for Typing and Read Receipts

**TL;DR:** For a multiplayer game lobby API, keep typing indicators out of durable room state and store read receipts as monotonic facts. Give them separate delivery paths even when they share one connection. The deciding constraint is fan-out: retrying every transient event multiplies work across a busy lobby, while dropping a receipt can leave another player with a permanently wrong unread marker.

This is the useful alternative-selection test for a Node.js multiplayer lobby: ignore the length of the feature list and ask whether the runtime lets each message class have the delivery guarantee it needs. A framework, hosted channel service, or small custom gateway can all fit. None fits if it forces durable room state, typing pulses, and receipts through one undifferentiated broadcast primitive.

## What should a multiplayer game lobby room state API guarantee?

A typing indicator describes the present moment. If `player-17` starts typing and the packet arrives after the player has stopped, delivery makes the interface worse. Put a short expiry on the signal, replace older signals from the same player, and let a missed update disappear. Do not replay a backlog of typing events after reconnect.

A read receipt carries different meaning. It says a player has observed chat through a stable message position. That fact should survive reconnects and process restarts. It also needs monotonic handling: receipt 184 may advance receipt 181, but a delayed 181 must never move the marker backward.

That split is small. It changes everything.

For a four-player room, broadcast logic can hide sloppy semantics. At 40 players, each input may create 39 outbound deliveries, and retries amplify that fan-out. The exact operating limit depends on payload size, update rate, network conditions, and runtime implementation, so measure it rather than borrowing somebody else's headline concurrency number.

## The simple broadcast experiment failed the meaning test

Start with the tempting design: receive an event, attach a room ID, then broadcast it to every socket in that room. It is easy to ship and easy to reason about during a local test. It also assigns the same failure policy to unrelated data.

The failure is clearest during reconnect. Replaying missed typing events produces stale UI. Refusing to replay anything loses receipts. Retrying the whole stream avoids one bug by creating another: obsolete transient messages sit in front of current state and spend bandwidth at precisely the moment a returning player needs a quick snapshot. The chosen boundary is semantic rather than transport-specific. Keep authoritative lobby membership and receipt cursors in server-owned storage. Keep typing state in memory with an expiry. On reconnect, send a current room snapshot plus the latest receipt cursor; resume live ephemeral delivery from now. WebRTC does not erase this decision. Its data channels can be configured for ordered or unordered delivery, and retransmission can be limited, but peers still need signaling outside the WebRTC specification. More important, transport knobs do not define who may claim that a message was read or how a reconnecting client catches up. Those remain application rules.

## A focused Node.js contract

The interface below keeps transport adapters replaceable and makes the guarantee visible at the call site. The gateway may be backed by sockets, peer data channels, or another realtime mechanism; business code does not get to pretend all events are equivalent.

```ts
type PlayerId = string;
type RoomId = string;

type TypingSignal = {
  kind: "typing";
  roomId: RoomId;
  playerId: PlayerId;
  active: boolean;
  expiresAtMs: number;
};

type ReadReceipt = {
  kind: "read";
  roomId: RoomId;
  playerId: PlayerId;
  throughSequence: number;
};

interface LiveFanout {
  publishLatest(key: string, signal: TypingSignal): Promise<void>;
  publishDurable(receipt: ReadReceipt): Promise<void>;
}

function acceptTyping(signal: TypingSignal, nowMs: number): boolean {
  return signal.expiresAtMs > nowMs;
}

function advanceReceipt(current: number, incoming: number): number {
  return Math.max(current, incoming);
}
```

`publishLatest` names the deliberate loss policy: a newer value supersedes an older one. `publishDurable` does not promise magical exactly-once delivery. It says the handler must tolerate duplicates, persist progress, and acknowledge only after the monotonic cursor is committed. A stable key such as `(roomId, playerId)` makes duplicate receipt updates harmless when the same `throughSequence` is retried.

Do not trust a client-supplied wall-clock timestamp to order receipts. The server should assign chat sequence numbers within the room, and the receipt should point to one of those positions. This also avoids tying correctness to synchronized player clocks.

## Selection hinges on fan-out behavior

Compare candidates with a workload, not a checkbox sheet. The candidate must expose enough control to separate replaceable messages from recoverable state. Then inspect where fan-out happens, how slow consumers are handled, and whether reconnect recovery is bounded.

A search for a Colyseus alternative often starts with API shape. This approach starts one level lower, with the observable result after loss, duplication, delay, and reconnect. That makes the comparison useful even when the shortlist changes.

Use a deliberately awkward test: several rooms of unequal size, one slow client, duplicate receipt submissions, an out-of-order receipt, a disconnect during a typing burst, and a reconnect after the typing expiry. Assert outcomes. The slow client must not stall unrelated rooms; old typing state must not return; receipt cursors must only advance; membership authorization must be checked before either event is accepted.

Three implementation families can pass that test. An authoritative multiplayer runtime may bundle room lifecycle and state synchronization. A general realtime channel service may reduce operational work but leave more domain rules in application code. A custom gateway gives direct control over queues and storage, while making capacity planning, reconnect logic, and on-call ownership your problem. The useful choice is the smallest operational burden that still exposes the two contracts.

There is a real limitation: splitting the paths adds two persistence policies, two sets of metrics, and more failure tests. It is a poor trade-off for a tiny lobby where receipts may disappear without consequence. In that case, one best-effort channel and a full snapshot on reconnect can be enough. The split becomes worthwhile when unread state must remain correct, rooms are large enough for retry fan-out to matter, or transient traffic is frequent enough to delay recovery data.

Cost follows the same shape. Count inbound events, fan-out deliveries, retained durable writes, reconnect reads, and peak concurrent connections. Typing frequency often dominates event count, while receipts dominate correctness work. Sampling, coalescing, and expiry can reduce transient load without weakening receipt integrity. Do not optimize both classes with the same lever.

## Measure this before copying the design

Measure end-to-end age for typing signals rather than delivery rate alone. A delivered signal that is already expired is a failure. Track dropped or superseded typing updates, outbound deliveries per inbound event, per-room queue depth, and the time a reconnect takes to receive its usable snapshot.

For receipts, track commit latency, duplicate submissions, rejected regressions, and the gap between the highest chat sequence displayed and the durable cursor. Test process termination immediately before and after persistence. That boundary reveals whether acknowledgement means accepted into memory or actually recoverable.

I would ship the split contract before tuning transports. It preserves the option to change the underlying runtime later, keeps transient chatter cheap, and makes the player-visible correctness rule testable. The decision is not which library broadcasts fastest in a demo. It is whether fan-out failure leaves the room briefly less animated or permanently wrong.

## References

- https://www.w3.org/TR/webrtc/
