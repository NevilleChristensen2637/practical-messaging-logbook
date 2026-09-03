# Node.js User Reminders: Cron, Postgres, and an Idempotent Queue Worker

Short answer: run cron every minute, move due reminders into a Postgres-backed job table in the same transaction, and let a separate worker deliver each notification with the reminder ID as its idempotency key.

Don't send notifications inside the cron process. Cron should only decide what is due. Delivery has different failure modes, latency, and retry rules, so tying both jobs together makes a slow notification provider hold the scheduler open and turns a routine retry into a duplicate-send risk.

For a developer tool that needs periodic cleanup or user reminders without keeping a web request open, this split is usually the smallest design worth shipping. It accepts up to roughly one minute of scheduling delay in exchange for low idle cost and very little machinery. Ship weekly; spend engineering time on the feature unless sub-minute delivery actually changes revenue.

## How much failure budget should Node.js cron, Postgres, and a queue worker get?

Treat scheduling and delivery as two state transitions. The scheduler changes a reminder from `pending` to `queued` and creates a durable job. The worker claims that job, calls the notification transport, and changes the reminder from `queued` to `sent`. A unique constraint on the job's `reminder_id` prevents two scheduler runs from creating two live jobs for one reminder.

The important detail is where that first transition happens. Inserting the job and updating the reminder belong in one database transaction. If code updates the reminder, commits, and then publishes to a separate queue, a process exit between those actions can leave a queued reminder with no job. Reversing the calls creates the opposite problem: a worker may see a job whose reminder never changed state. An outbox-style job table in the same Postgres database closes that gap without adding a coordinator.

Write the failure windows down before choosing a queue. A scheduler can exit after reading due rows but before claiming them. Two cron processes can select the same rows during a deploy. A worker can lose its lease during a slow send. Most dangerously, the transport can accept a notification just before the worker loses the acknowledgement. Each window needs a different defense; calling all four of them “retries” hides the design work.

This is the state machine I would put in a build log:

| State | Owner | Allowed next state | Meaning |
| --- | --- | --- | --- |
| `pending` | scheduler | `queued` | Waiting for `due_at` |
| `queued` | worker | `sent` or retry | Durable job exists |
| `sent` | nobody | none | Delivery accepted |

Keep the cron handler short. Two instances may overlap during a deploy, so the query must claim rows rather than merely read them. `FOR UPDATE SKIP LOCKED` lets concurrent schedulers work on different eligible rows; the unique job key is a second guard, not the primary locking strategy. Use database time for the comparison so application hosts with slightly different clocks don't disagree about which reminders are due.

The one-minute cadence sets the first latency bound. A reminder created just after a tick waits almost a full interval before it can be queued, then adds worker wait time and provider latency. That's acceptable for most reminder emails and many push notifications. It is not suitable for second-sensitive alerts, interactive timers, or authentication codes. Those need a scheduler with finer wakeups, a delayed-message primitive, or a dedicated timing service.

## Idempotency policy and dead-letter governance

A database transaction cannot cover an external notification provider. Consider the narrow failure window: the provider accepts a message, then the worker exits before it records `sent`. The lease expires and another worker retries. From Postgres alone, that retry is indistinguishable from a request the provider never accepted.

No clever status column fixes this.

Pass the stable reminder ID as an idempotency key when the transport supports one. Reusing that key on retries lets the transport recognize the same logical delivery. If the transport has no idempotency facility, the system is at-least-once and users can receive duplicates. Be honest about that contract. For email, a product may tolerate a rare duplicate; for billing notices or destructive automation, use a transport or downstream protocol that can deduplicate.

Also define what “accepted” means. It usually means the transport accepted responsibility, not that a person read the message. Delivery receipts, bounces, and user interaction are separate events and should not reopen the scheduling job. Mixing those lifecycles creates jobs that never reach a stable terminal state.

Retries need a ceiling. Move a repeatedly failing job to a dead-letter table after a configured attempt count, preserving its payload, last error category, and timestamps for inspection. Dead-letter queues isolate work that cannot be processed successfully, but their retention and redrive policy still require an operator decision. Don't create an infinite hot loop.

Rate limiting is different from a permanent failure. HTTP `429 Too Many Requests` means the client exceeded a rate limit, and a server may include `Retry-After` to indicate when another request can be made. Honor that value when present; otherwise use bounded exponential backoff with jitter. A malformed address or revoked destination belongs in a non-retryable category. Classifying both cases as “send failed” wastes capacity and delays healthy reminders.

## Build log: integrate the transactional claim path

The schema needs two durable records: the user's intent and the delivery work. The example assumes `reminders.id` and `notification_jobs.reminder_id` are UUID columns, `due_at` is a timezone-aware timestamp, and the job table has a unique constraint on `reminder_id`. Index pending reminders by state and due time, and index available jobs by availability and lease time. Those indexes keep each poll aimed at the small set of actionable rows as history grows.

The scheduler below receives a generic transaction-capable database interface. There is no queue SDK because Postgres is the queue at this scale. That keeps the undifferentiated part easy to replace later.

```ts
type QueryResult<T> = { rows: T[] };

interface DbClient {
  query<T>(sql: string, values?: unknown[]): Promise<QueryResult<T>>;
}

interface Database {
  transaction<T>(work: (client: DbClient) => Promise<T>): Promise<T>;
}

export async function enqueueDueReminders(db: Database): Promise<number> {
  return db.transaction(async (client) => {
    const inserted = await client.query<{ reminder_id: string }>(
      `WITH due AS (
         SELECT id, user_id, payload
         FROM reminders
         WHERE status = 'pending' AND due_at <= CURRENT_TIMESTAMP
         ORDER BY due_at, id
         FOR UPDATE SKIP LOCKED
         LIMIT 100
       ), created AS (
         INSERT INTO notification_jobs
           (reminder_id, payload, available_at, attempts, leased_until)
         SELECT id, payload, CURRENT_TIMESTAMP, 0, NULL
         FROM due
         ON CONFLICT (reminder_id) DO NOTHING
         RETURNING reminder_id
       )
       UPDATE reminders AS r
       SET status = 'queued'
       FROM created
       WHERE r.id = created.reminder_id
       RETURNING created.reminder_id`,
    );

    return inserted.rows.length;
  });
}
```

Run that function from a cron entry once per minute. Do not accept a `due_at` cutoff from the incoming web request; the web request should persist the reminder and return. The scheduled process owns discovery of due work.

The worker uses a lease so an interrupted process does not own a job forever. Each claim increments `attempts` and sets `leased_until`. A successful delivery marks the reminder sent and removes the job in one transaction. A failed attempt changes `available_at` for a later retry and clears the lease. Keep the delivery call outside the database transaction because network I/O can be slow.

```ts
type Job = {
  reminder_id: string;
  payload: unknown;
  attempts: number;
};

interface NotificationTransport {
  send(input: {
    payload: unknown;
    idempotencyKey: string;
  }): Promise<{ accepted: boolean; retryAfter?: Date }>;
}

async function claimOne(client: DbClient): Promise<Job | undefined> {
  const result = await client.query<Job>(
    `WITH candidate AS (
       SELECT reminder_id
       FROM notification_jobs
       WHERE available_at <= CURRENT_TIMESTAMP
         AND (leased_until IS NULL OR leased_until < CURRENT_TIMESTAMP)
       ORDER BY available_at, reminder_id
       FOR UPDATE SKIP LOCKED
       LIMIT 1
     )
     UPDATE notification_jobs AS j
     SET leased_until = CURRENT_TIMESTAMP + INTERVAL '30 seconds',
         attempts = attempts + 1
     FROM candidate
     WHERE j.reminder_id = candidate.reminder_id
     RETURNING j.reminder_id, j.payload, j.attempts`,
  );
  return result.rows[0];
}

export async function workOnce(
  db: Database,
  transport: NotificationTransport,
): Promise<boolean> {
  const job = await db.transaction(claimOne);
  if (!job) return false;

  const result = await transport.send({
    payload: job.payload,
    idempotencyKey: job.reminder_id,
  });

  if (!result.accepted) {
    const retryAt = result.retryAfter ?? new Date(Date.now() + 60_000);
    await db.transaction((client) =>
      client.query(
        `UPDATE notification_jobs
         SET available_at = $2, leased_until = NULL
         WHERE reminder_id = $1`,
        [job.reminder_id, retryAt],
      ).then(() => undefined),
    );
    return true;
  }

  await db.transaction(async (client) => {
    await client.query(
      `UPDATE reminders SET status = 'sent' WHERE id = $1`,
      [job.reminder_id],
    );
    await client.query(
      `DELETE FROM notification_jobs WHERE reminder_id = $1`,
      [job.reminder_id],
    );
  });
  return true;
}
```

Thirty seconds is an example lease, not a universal setting. Set it longer than the transport's normal request timeout, then renew it if one delivery can legitimately take longer. The same goes for the batch limit of 100. I'm not sure what either number should be for a new workload until its delivery latency and due-volume distribution are visible; start bounded, measure, and change one knob at a time.

## Roll out with concurrency tests

Test the state machine, not cron itself. Freeze database time in an integration database; insert reminders just before, at, and just after the cutoff; run two schedulers concurrently; and assert that each reminder has at most one job. Then interrupt a worker after claim, let the lease expire, and verify that the job becomes claimable again. Finally, simulate an accepted delivery followed by loss of the local acknowledgement and confirm that the same idempotency key is reused.

In production, graph oldest due reminder age, oldest available job age, claim rate, retry count by category, dead-letter count, and delivery latency. Queue depth alone can look healthy while one poisoned partition starves. Alert on user-visible age against the product promise. Logs should carry `reminder_id`, attempt number, and a transport request identifier, but not the notification body.

## Latency and cost decide the migration

Postgres polling is a good fit when reminder traffic is moderate, the database is already operated, and one-minute resolution meets the product promise. It minimizes services, credentials, deployment paths, and invoices. For a one-person SaaS, that operational compression often matters more than shaving a few seconds from background work. The revenue-per-hour question is blunt: will another queue system help ship or retain enough users to pay for its integration and on-call surface?

The catch is database contention. As volume grows, scheduler batches become continuously full, worker leases churn through many row updates, and old job records compete with application traffic. At that point, partition work by a stable key, increase worker concurrency carefully, and watch claim latency rather than only queue depth. If background traffic starts threatening foreground queries, move delivery jobs to a dedicated queue and keep a transactional outbox in Postgres to publish them. The outbox remains necessary because the cross-system write still cannot be atomic by wishful thinking.

Stick with the database design when simplicity and low idle cost dominate. Choose a dedicated queue when independent scaling, built-in redrive, longer retention, or isolation from the primary database is worth another dependency. Choose a specialized scheduler when reminders must fire much closer than one minute or when the future schedule is too large to poll efficiently. Those are capability boundaries, not maturity badges.

Start small. Keep the boundary clean.

## Further reading

- [Dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
