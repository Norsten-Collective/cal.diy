# Norsten operator notes: environment variables not in upstream examples

This fork (`cal.norstenco.com`) is otherwise vanilla upstream cal.diy. The root `.env.example` and
`.env.appStore.example` cover almost everything. This file documents only the gap found in the
Norsten batch 1 audit. It does not change runtime behaviour.

## `CRON_SECRET`

```bash
# Server-only shared secret; cron callers must send it as "Authorization: Bearer <value>"
CRON_SECRET=
```

- **Side:** server only. Never prefix with `NEXT_PUBLIC_` and never expose to the browser.
- **Status:** current, used in source, missing from upstream `.env.example`.
- **Operator action:** generate a strong random value (for example `openssl rand -base64 32`) and set it
  on whichever host runs the web app, and on whatever scheduler calls the cron endpoints. Store it in
  1Password. Do not commit it.

### Where it is checked

| Endpoint | Handler | Accepted auth | Without `CRON_SECRET` set |
|---|---|---|---|
| `/api/tasks/cron` | `packages/features/tasker/api/cron.ts` | `Bearer <CRON_SECRET>` only | **Required.** No other credential is accepted |
| `/api/tasks/cleanup` | `packages/features/tasker/api/cleanup.ts` | `Bearer <CRON_SECRET>` only | **Required.** No other credential is accepted |
| `/api/cron/calendar-subscriptions` | `apps/web/app/api/cron/calendar-subscriptions/route.ts` | `CRON_API_KEY` (header or `?apiKey=`) **or** `Bearer <CRON_SECRET>` | Optional if the caller uses `CRON_API_KEY` |
| `/api/cron/calendar-subscriptions-cleanup` | `apps/web/app/api/cron/calendar-subscriptions-cleanup/route.ts` | same as above | Optional if the caller uses `CRON_API_KEY` |
| `/api/cron/selected-calendars` | `apps/web/app/api/cron/selected-calendars/route.ts` | same as above | Optional if the caller uses `CRON_API_KEY` |

`apps/web/cron-tester.ts` (local dev helper) also sends it.

The schedules for these paths are in `apps/web/vercel.json`. `/api/tasks/cron` is scheduled every minute.
If nothing calls it with the right header, queued tasker jobs are not processed.

### Why it must be set even if unused

The handlers compare the header against the string `` `Bearer ${process.env.CRON_SECRET}` ``. If the variable
is unset, that string becomes `Bearer undefined`, which is a guessable value. Always set a strong secret on
every deployment, even where another credential is used.

## Related check: `CRON_API_KEY`

The upstream `.env.example` ships `CRON_API_KEY` with a non-empty template literal. Any deployment must
override it with its own strong secret. The template value is public in this repository.
