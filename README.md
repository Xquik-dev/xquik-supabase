# X (Twitter) Data & API in Supabase with Xquik

Use this Supabase toolkit by Xquik to read X (Twitter) data from SQL.
Edge Functions call the same API, and a receiver stores signed webhook events.

Xquik is the world's fastest and cheapest X (Twitter) scraper service with the most complete X (Twitter) data.
Most other Apify Actors charge before filtering or deduplication.
Xquik charges only for delivered, unique, filter-matching results and never pads rows.
Xquik is independent and not affiliated with X Corp.

<!-- trust:win:start -->

The Xquik REST API read 45 tweets per second at $0.15 per 1,000 on 2026-10-15. The fastest of 4 rival Actors read 12.5 per second. The cheapest cost $0.173 per 1,000. [Compare each run](https://xquik.com/benchmarks#benchmark-x-tweet-scraper).

<!-- trust:win:end -->

## Install the Supabase X (Twitter) toolkit

You need the [Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started) & Docker.
Get an API key & a webhook secret from your [Xquik account](https://xquik.com).

```bash
git clone https://github.com/Xquik-dev/xquik-supabase.git
cd xquik-supabase
XQUIK_API_KEY=xq_your_key XQUIK_WEBHOOK_SECRET=your_webhook_secret npx supabase start
```

The CLI also reads both values from a `.env` file in the project root.
Git ignores that file.

## Call the X (Twitter) API from SQL

Open the SQL editor of your local Studio and run these lines.
`xquik.call` returns the JSON answer.
`xquik.rows` returns 1 row per list item.
The second query lists the latest tweets of 1 account.

```sql
select xquik.call('getUser', '{"id": "NASA"}');

select tweet ->> 'text'
from xquik.rows('searchTweets', '{"q": "from:NASA"}') as tweet;
```

Next, read the [Xquik Supabase guide](https://docs.xquik.com/guides/supabase), the [foreign tables](#query-x-twitter-data-as-foreign-tables) or the [webhook receiver](#receive-x-twitter-webhook-events).

<!-- trust:start -->

## Benchmarks & usage

[![Twitter API p99 server read time](https://img.shields.io/endpoint?url=https%3A%2F%2Fxquik.com%2Fbadges%2Frest-p99.json)](https://xquik.com/benchmarks#speed)
[![Twitter API package downloads](https://img.shields.io/endpoint?url=https%3A%2F%2Fxquik.com%2Fbadges%2Fdownloads.json)](https://xquik.com/benchmarks#downloads)

- 6.4 s p99 REST API server read time over 7 days, failed reads included ([source](https://xquik.com/benchmarks#speed)) <!-- trust:rest-p99 6317 -->
- 63,000+ downloads of Xquik's Twitter API packages on npm, PyPI, RubyGems, NuGet, Packagist & Terraform ([source](https://xquik.com/benchmarks#downloads)) <!-- trust:downloads 63178 -->

Checked on 2026-11-06. See each number with its date & source on [xquik.com/benchmarks](https://xquik.com/benchmarks) or as [JSON](https://xquik.com/trust.json).

<!-- trust:end -->

## Call more operations

Path parameters such as `id` go into the URL.
Other parameters go into the query string for GET & DELETE.
They go into the JSON body for other methods.
Run `select * from xquik.operations` to list every operation id.

Only the service role runs these functions.
Call them from SQL, cron jobs, triggers or a server with that role's key.

## Call Xquik from an Edge Function

`createXquik()` reads `XQUIK_API_KEY` & returns the typed client.

```ts
import { createXquik } from "../_shared/xquik.ts";

const xquik = createXquik();

Deno.serve(async () => {
  const nasa = await xquik.x.users.retrieve("NASA");
  return Response.json(nasa);
});
```

The client types every method & parameter.
A function that imports `_shared/xquik.ts` lists `x-twitter-scraper` and `zod` in its own `deno.json`.

## Query X (Twitter) data as foreign tables

Each public JSON read is a foreign table in the `xquik` schema.
Its name is the operation id in snake case.

```sql
select attrs ->> 'name', attrs ->> 'followers' from xquik.get_user where id = 'NASA';

select attrs ->> 'text' from xquik.search_tweets where q = 'from:NASA' limit 20;
```

- Path & query parameters are columns, & an `=` filter sends them.
- Quote mixed-case names, as in `where "pageSize" = 50`.
- `attrs` holds each row as JSON.
- A SQL `limit` clause stops paging early.
- A query reads at most 10 pages until you raise the server's `max_pages`.
- Raise it with `alter server xquik_server options (set max_pages '50')`.
- The server keeps its options when the tables change.
- The tables read your API key from the `xquik_api_key` Vault secret.
- Query them as `postgres`, the role the SQL editor & cron jobs use.
- `xquik.export_draw` & `xquik.export_extraction` return JSON rows.
- Event streams & support attachment downloads have no table.
- Use the `xquik-webhook` receiver or an Edge Function for them.

When `openapi.json` changes, the next migration rebuilds only the changed tables.
It drops the tables of removed operations.
It also drops any other foreign table on `xquik_server` in the `xquik` schema.
A view on a rebuilt or dropped table stops the migration.
Drop the view, run the migration & create the view again.

## Schedule calls

`xquik.schedule` creates a pg_cron job that adds an action on each run.
Run it again with the same name to change the job.

```sql
select xquik.schedule('nasa-tweets', '*/15 * * * *', 'getUserTweets', '{"id": "NASA"}');
```

## Run actions from rows

Insert a row into `public.xquik_actions` from your app, a trigger or a Database Webhook.
pg_net sends it after your transaction commits.
The `xquik-collect` cron job copies finished answers every 10 seconds.

```sql
select xquik.enqueue('getUser', '{"id": "NASA"}');

select status, http_status, response
from public.xquik_actions
order by id desc
limit 1;
```

`status` moves from `sent` to `succeeded` or `failed`.
pg_net sends only GET, POST & DELETE.
Run PATCH & PUT operations with `xquik.call`.

## Receive X (Twitter) webhook events

The `xquik-webhook` function stores each event in `public.xquik_events`.
It checks the signature, the timestamp & the nonce first.
It writes with the secret key named `default`.
It keeps working after you turn off the legacy API keys.

- Every event type & the test alert get a 200.
- A retry of a stored event gets a 200 and adds no row.
- A bad signature, an old timestamp or a reused nonce gets a 401.
- A storage error gets a 503, so Xquik retries the delivery.

Read events in order with `order by occurred_at, stream_event_id`.

```sql
select *
from public.xquik_events
order by occurred_at, stream_event_id
limit 10;
```

Put slow work in an `after` callback.
Pass it to `handleDelivery` in `supabase/functions/xquik-webhook/index.ts`.
Wrap it in `EdgeRuntime.waitUntil`.
It runs after Xquik gets its answer, once per new event.

Each webhook URL has its own secret.
Set several secrets in `XQUIK_WEBHOOK_SECRET` with commas between them.
This also lets you rotate a secret without missing events.
A pg_cron job removes nonces older than 10 minutes.

## Keep tables private

Row level security covers every regular table.
Policies deny `anon` & `authenticated` each operation, and neither holds a grant.
Only the service role reads these tables.

## Deploy to a hosted project

```bash
supabase link --project-ref your-project-ref
supabase db push
supabase secrets set XQUIK_API_KEY=xq_your_key XQUIK_WEBHOOK_SECRET=your_webhook_secret
supabase functions deploy xquik-webhook
```

Then save the API key in Vault for SQL calls.
Run this once in the SQL editor.

```sql
select vault.create_secret('xq_your_key', 'xquik_api_key');
```

Your webhook URL is `https://your-project-ref.supabase.co/functions/v1/xquik-webhook`.
`supabase/config.toml` turns off the JWT check for this function.
The signature check replaces it.

## Questions

### Do I need an X (Twitter) developer account?

No. Public reads need only an Xquik API key.
Private reads & account actions need a connected X (Twitter) account.

### Is this the official X (Twitter) API?

No. Xquik runs its own REST API.
This toolkit calls that API, not the official X (Twitter) API.

## Develop

```bash
deno task check
deno task test
XQUIK_WEBHOOK_SECRET=your_webhook_secret deno task test:stack
supabase test db
deno task generate
deno task types
```

`deno task test` proves that every operation in `openapi.json` has a tested path.
It also proves that every event type gets a receiver result.
`deno task test:stack` runs against `supabase start`.
`supabase test db` runs the pgTAP tests in `supabase/tests/database`.
`deno task generate` rebuilds the operation list & the foreign tables from `openapi.json`.
`deno task types` regenerates `supabase/functions/_shared/database.types.ts`.
`bin/check-local` installs the pinned Deno with Bun, then runs the checks & tests.

## Documentation

- [Xquik Supabase guide](https://docs.xquik.com/guides/supabase)
- [Xquik API reference](https://docs.xquik.com/api-reference/overview)
- [Webhook signature verification](https://docs.xquik.com/webhooks/verification)
- [Organization support policy](https://github.com/Xquik-dev/.github/blob/main/SUPPORT.md)
- [Security policy](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## More Xquik tools

- [X (Twitter) scraper API & MCP server](https://github.com/Xquik-dev/x-twitter-scraper) with REST, MCP & Skill setup.
- X (Twitter) API SDKs for [TypeScript](https://github.com/Xquik-dev/x-twitter-scraper-typescript), [Python](https://github.com/Xquik-dev/x-twitter-scraper-python), [Go](https://github.com/Xquik-dev/x-twitter-scraper-go), [Ruby](https://github.com/Xquik-dev/x-twitter-scraper-ruby), [Java](https://github.com/Xquik-dev/x-twitter-scraper-java), [Kotlin](https://github.com/Xquik-dev/x-twitter-scraper-kotlin), [C#](https://github.com/Xquik-dev/x-twitter-scraper-csharp) & [PHP](https://github.com/Xquik-dev/x-twitter-scraper-php).
- [X (Twitter) CLI](https://github.com/Xquik-dev/x-twitter-scraper-cli) for terminal workflows.
- The [X (Twitter) API alternative skill](https://github.com/Xquik-dev/x-api) & [X (Twitter) advanced search operators](https://github.com/Xquik-dev/x-twitter-advanced-search).
- X (Twitter) agent plugins for [OpenClaw](https://github.com/Xquik-dev/tweetclaw) & [Hermes Agent](https://github.com/Xquik-dev/hermes-tweet).
- The [X (Twitter) API documentation source](https://github.com/Xquik-dev/xquik-docs).
- X (Twitter) integrations for [n8n](https://github.com/Xquik-dev/n8n-nodes-xquik), [Prefect](https://github.com/Xquik-dev/prefect-xquik), [Haystack](https://github.com/Xquik-dev/xquik-haystack), [Paperclip](https://github.com/Xquik-dev/paperclip-plugin-xquik), [Terraform](https://github.com/Xquik-dev/terraform-provider-x-twitter-scraper), [Convex](https://github.com/Xquik-dev/xquik-convex) & [Strapi](https://github.com/Xquik-dev/strapi-plugin-xquik).
- X (Twitter) MCP for [AI agents](https://github.com/Xquik-dev/xquik-mcp), [VS Code](https://github.com/Xquik-dev/xquik-vscode) & [Raycast](https://github.com/Xquik-dev/xquik-raycast).
- [Xquik X (Twitter) API & scraper](https://xquik.com) and the [X (Twitter) API documentation](https://docs.xquik.com/api-reference/overview).

Xquik is an independent third-party service. Not affiliated with X Corp. "Twitter" and "X" are trademarks of X Corp.
