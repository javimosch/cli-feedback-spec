---
name: add-feedback-command
description: Add a `feedback` command to an agent-first CLI that dual-writes to the app's own /v1/feedback endpoint and a central feedback relay. Use when asked to "add feedback", "wire up feedback", "hook into the feedback relay", or give a tool a way to report bugs/ideas.
---

# Add a feedback command (feedback-spec convention)

Give a CLI a `feedback` subcommand that goes two places at once: the tool's own endpoint
(each app owns its feedback) and a central relay (one inbox across every tool). Full norms
in [PROTOCOL.md](PROTOCOL.md); full how-to in [RECIPE.md](RECIPE.md). This skill is the
checklist.

## Decide the shape first

- **Server + CLI** (the tool runs a daemon): implement all four steps below.
- **Relay-only** (a thin client / peer tool with no server): skip step 2; in step 3 do only
  the relay write. See `remotecmd` (Go) as the worked example.

## The four edits

1. **Storage table** in your migrations:
   `feedback(id TEXT PRIMARY KEY, kind, message, context, reporter, version, created INTEGER, ip)`.
   Idempotency = the primary key on `id`.

2. **Endpoint** `POST /v1/feedback`: open, `message` required (else 400), body cap 16KB
   (else 413), `INSERT OR IGNORE` on `id`, record client IP, return
   `{"ok":true,"id":"…","stored":true}`. Copy the handler from RECIPE.md step 2 and route it.

3. **CLI `feedback` command** — the dual-write, the important part:
   - `myapp feedback "<message>" [-kind bug|idea|praise|note] [-context "…"]`
   - build ONE JSON body with a fresh random `id`, `app`+`version` tags, `reporter` = `$USER`
     or `agent`;
   - POST to the app endpoint (`MYAPP_URL` → `MYAPP_PUBLIC_URL` → compiled default);
   - POST to the relay (`FEEDBACK_RELAY`, default `https://feedback.intrane.fr`, `off` skips);
   - **same `id` on both** (idempotent); **never fail the caller**; print
     `{"ok":true,"data":{"id":"…","stored":0|1,"relayed":0|1}}`.
   - Dispatch `feedback` **before** opening any local DB, so relay-only submits work with no
     local state.

4. **Document + bump**: add a short "Feedback" section to the README (name the command,
   `FEEDBACK_RELAY`, and the app URL env var); bump the version.

## Verify before you're done

```sh
curl -s -XPOST localhost:PORT/v1/feedback -d '{"message":"hi","id":"t1"}'   # {"ok":true,"id":"t1","stored":true}
curl -s -XPOST localhost:PORT/v1/feedback -d '{"kind":"bug"}'               # 400 message required
MYAPP_URL=http://localhost:PORT FEEDBACK_RELAY=off myapp feedback "test" -kind bug   # stored:1 relayed:0
# live: confirm the relay saw your app
curl -s "https://feedback.intrane.fr/v1/feedback?app=myapp" -H "authorization: Bearer $FEEDBACK_ADMIN_TOKEN"
```

## Gotchas

- Build outbound JSON with a real encoder (machin `json(x)`), never string-concat raw user
  text.
- Prefix helpers that collide with your web framework (`client_ip`, `route`).
- If you self-host the relay, its data dir must be writable by the daemon user — otherwise it
  200s without persisting.
