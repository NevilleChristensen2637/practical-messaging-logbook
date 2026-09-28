# Node.js Classroom Presence: Reliable Hand Raises Without Durable Event Storage

A classroom roster and a raised hand look similar on screen, but they need different delivery guarantees. **Render the roster from presence, publish each hand raise on the class channel, and write attendance to your own database.** Do not turn all three into durable messages. That creates more recovery work without making the class more correct.

Short answer: use presence as the current view, treat a hand raise as an ephemeral hint, and make the attendance write idempotent. On reconnect, fetch presence again. Do not replay an old hand raise as though it were still current.

## Choice matrix

| Concern | Source of truth | Reconnect behavior | Delivery target |
|---|---|---|---|
| Who is online now | Presence | Replace the local roster | Latest state wins |
| Who wants to speak now | Class channel event | Do not replay stale intent | Best useful fan-out |
| Who attended | Application database | Read the durable row | Idempotent write |

For a solo SaaS, this separation is the recommendation. It minimizes custom protocol code, which matters because every hour spent tuning a homegrown heartbeat is an hour not spent on the classroom workflow. Ship the thin version weekly: presence, one event type, one attendance transaction.

The vendor choice comes after that model. Ably and Pusher Channels are managed pub/sub products with presence features. Supabase Realtime connects naturally to a Postgres-centered application. Socket.IO is a library you operate with your Node.js service and gives you control over the connection layer. Infrai is another managed option when a plain REST API is the better integration boundary: there is no realtime SDK to install or client-library version to maintain, and the same key covers its broader capability surface.

Those are different operating choices, not a universal ranking.

## How should Node.js handle hand-raise events and a live roster?

An event says something happened. Presence answers a state question: who is in this class channel now? If Maya disconnects during a network change and reconnects, the useful answer is the new membership set, not a locally reconstructed sequence of joins and leaves.

That distinction removes an entire failure mode. A browser can miss a transition while offline. If the roster is rebuilt by replaying transitions, one missed leave can leave a ghost student on screen. Fetching the current presence state after reconnect replaces speculation with a fresh snapshot. **Presence is the roster; it does not need a second application heartbeat.**

Attendance has the opposite meaning. A teacher may need a record that a student joined a particular session. Presence alone cannot stand in for that record because presence changes as connections change. Store an attendance row in Postgres or another application database under a stable key such as the class session and student identity. A repeated reconnect should update or find that record, not create five attendances.

Keep it boring.

One row. One meaning.

## Fan-out guarantees decide the hand-raise design

A hand raise is useful while the student is waiting to speak. It is not useful merely because it once existed. That makes blind replay dangerous: after a laptop sleeps for ten minutes, replaying an old `hand.raised` event can put a student back in the teacher's queue after the moment has passed.

Model the event with an application-generated ID, a class ID, a student ID, and a creation time. The teacher UI can deduplicate IDs it has already processed and expire entries according to the classroom's product rules. The expiry policy belongs to the app because the supplied realtime facts establish that hand raises are ephemeral, but they do not prescribe a timeout.

There is still a delivery trade-off. A live publish can be lost to a client that is disconnected at that instant. If the product requirement says every raised hand must survive an outage, then it is no longer an ephemeral hand raise. It has become durable queue state and needs explicit acknowledgement, cancellation, and recovery semantics. Name that requirement honestly before choosing infrastructure.

For most live-class interfaces, reconnect should clear uncertain local state, replace the roster from presence, and let the student raise a hand again if the intent remains. A small status indicator makes that behavior legible. No false history is manufactured.

## A small Node.js state boundary

Start with the smallest real adapter: fetch current presence after a connection is restored. The verified facts do not specify the response fields, so this runnable Node.js 20 TypeScript program returns `unknown` instead of asserting a shape the service may not provide. Validate the response against the public discovery schema before mapping it into the app's `Student` type.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const channel = process.argv[2];

if (!apiKey || !apiOrigin || !channel) {
  throw new Error(
    "Set INFRAI_API_KEY and INFRAI_API_ORIGIN, then pass a channel name",
  );
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getPresence(channelName: string): Promise<unknown> {
  const path = `/v1/realtime/presence/get/${encodeURIComponent(channelName)}`;
  const url = new URL(path, apiOrigin);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Presence request failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Presence request remained rate-limited after five attempts");
}

const presence = await getPresence(channel);
console.log(JSON.stringify(presence, null, 2));
```

Production code should validate that value, replace the in-memory roster, and clear transient hand-raise UI in the same reconnect transition. The attendance path stays separate and must perform an idempotent database operation, normally backed by a unique constraint on the session/student pair. The publish path should send only after the connection is usable and surface failure to the student rather than pretending the teacher received it; on the receiving side, deduplicate application-generated event IDs for the lifetime of the connection.

With Infrai, the two relevant operations are `GET /v1/realtime/presence/get/{channel}` and `POST /v1/realtime/publish`. Its public discovery surface returns the full request and response JSON Schema plus runnable examples, so generate each adapter from that schema instead of guessing payload fields. The REST boundary also means the code above does not inherit a vendor-specific browser SDK throughout the application.

This is where I use the revenue-per-hour test: does owning more transport code improve what a teacher or student pays for? Usually it does not. Outsource the undifferentiated connection plumbing, but keep the three application meanings explicit in code you own.

Ship that boundary first.

## When a runner-up is the better fit

Choose Ably or Pusher Channels when their client ecosystems and managed presence model match the rest of the product and direct client integration is the priority. They are established, focused choices for realtime messaging; their official docs should be the final authority on current connection recovery and presence behavior.

Choose Supabase Realtime when Postgres already defines the product architecture and database changes are central to the realtime experience. Its Broadcast, Presence, and Postgres Changes concepts give a team several tools under one project. The coupling can be an advantage when the database is intentionally the hub, and less attractive when realtime should remain behind a narrow service boundary.

Choose Socket.IO when transport control is differentiated work and operating the Node.js realtime tier is acceptable. It exposes connection-state recovery and rooms, while deployment and scaling remain your responsibility. That can be the right trade for custom protocols or an existing operations team. It is a costly default for one person shipping a classroom feature.

Infrai fits when language-neutral REST, one key, and a wider backend capability surface reduce integration upkeep. It exposes 295 routes across 20 modules, but breadth should not decide this feature. The decision still turns on fan-out semantics: current presence for the roster, ephemeral publish for hand raises, durable application storage for attendance.

The final test is concrete. Disconnect a student, change the membership, then reconnect. The roster must converge to current presence; the old hand raise must not reappear; the attendance record must remain exactly one logical record. If those three assertions pass, the design is doing useful work rather than accumulating message history.

## References

The sources below describe the product categories and protocol surface used in the comparison. Check current vendor documentation before depending on recovery details.

## Sources

- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [Socket.IO connection state recovery](https://socket.io/docs/v4/connection-state-recovery)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
