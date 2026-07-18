# Recipe — add a `feedback` command to a CLI

This is the how-to companion to [PROTOCOL.md](PROTOCOL.md). It's the exact pattern the
[reference implementations](README.md#reference-implementations) follow. Four edits; ~30
minutes; language-neutral.

If your tool is a **thin client / peer tool with no server of its own**, skip step 2 and
implement only the relay write in step 3 (the "relay-only" shape — see `remotecmd`).

---

## Step 1 — a storage table

Wherever your app runs its schema/migrations, add:

```sql
CREATE TABLE IF NOT EXISTS feedback(
  id TEXT PRIMARY KEY, kind TEXT, message TEXT, context TEXT,
  reporter TEXT, version TEXT, created INTEGER, ip TEXT
);
```

Idempotency lives in the `PRIMARY KEY` on `id`.

## Step 2 — the endpoint `POST /v1/feedback`

Open, size-capped, `message` required, `INSERT OR IGNORE` on `id`, record the client IP.
Return `{"ok":true,"id":"…","stored":true}`. In [machin](https://github.com/javimosch/machin):

```go
func h_feedback(req) (res) {
    if len(req.body) > 16384 { res = response("413 Payload Too Large", "application/json", "{\"ok\":0,\"error\":\"feedback too large\"}")  return }
    mv, merr := json_get(req.body, ".message")
    msg := ""
    if merr == "" { msg = trim(unq(mv)) }
    if msg == "" { res = bad_request("message required")  return }
    db := db_open()
    db_migrate(db)
    idv, _ := json_get(req.body, ".id")
    id := unq(idv)
    if id == "" { id = to_hex(rand_bytes(16)) }
    kv, _ := json_get(req.body, ".kind")
    cv, _ := json_get(req.body, ".context")
    rv, _ := json_get(req.body, ".reporter")
    vv, _ := json_get(req.body, ".version")
    sqlite_exec(db, "INSERT OR IGNORE INTO feedback(id, kind, message, context, reporter, version, created, ip) VALUES(?,?,?,?,?,?,?,?)",
        []string{id, unq(kv), msg, unq(cv), unq(rv), unq(vv), str(now()), client_ip(req)})
    sqlite_close(db)
    res = ok_json("{\"ok\":true,\"id\":\"" + id + "\",\"stored\":true}")
}
```

Then route it:

```go
if req.method == "POST" && p == "/v1/feedback" { res = h_feedback(req)  return }
```

Rate limiting: reuse whatever per-IP limiter your app already has; if none, the central
relay still rate-limits its own intake.

## Step 3 — the CLI `feedback` command (the dual-write)

The heart of the convention. Build one body, POST it to your endpoint and to the relay,
never fail the caller. In machin:

```go
func fb_post(url, payload) (ok) {
    ok = 0
    status, _, err := http_request("POST", url, []string{"content-type: application/json"}, payload)
    if err == "" && status >= 200 && status < 300 { ok = 1 }
}

func cmd_feedback(a) {
    if len(a) < 3 || a[2] == "" { fail(80, "usage: myapp feedback \"<message>\" [-kind bug|idea|praise] [-context \"...\"]") }
    msg := a[2]
    kind := opt(a, "-kind", "note")
    ctx := opt(a, "-context", "")
    reporter := env("USER")
    if reporter == "" { reporter = "agent" }
    id := to_hex(rand_bytes(16))
    payload := "{\"app\":\"myapp\",\"version\":" + json(app_version()) + ",\"kind\":" + json(kind) + ",\"message\":" + json(msg) + ",\"context\":" + json(ctx) + ",\"reporter\":" + json(reporter) + ",\"id\":" + json(id) + "}"
    app := env("MYAPP_URL")
    if app == "" { app = env("MYAPP_PUBLIC_URL") }
    if app == "" { app = "https://myapp.example.com" }
    stored := fb_post(app + "/v1/feedback", payload)
    relay := env("FEEDBACK_RELAY")
    if relay == "" { relay = "https://feedback.intrane.fr" }
    relayed := 0
    if relay != "off" { relayed = fb_post(relay + "/v1/feedback", payload) }
    out_ok("{\"id\":" + json(id) + ",\"stored\":" + str(stored) + ",\"relayed\":" + str(relayed) + "}")
}
```

Same idea in **Go** (the relay-only shape — no local endpoint, so just the relay write):

```go
func handleFeedback(args []string) {
    msg := args[0] // + parse -kind/-context
    idb := make([]byte, 8); rand.Read(idb); id := hex.EncodeToString(idb)
    reporter := os.Getenv("USER"); if reporter == "" { reporter = "agent" }
    payload, _ := json.Marshal(map[string]string{
        "app": "myapp", "version": Version, "kind": kind, "message": msg,
        "context": context, "reporter": reporter, "id": id,
    })
    relay := os.Getenv("MYAPP_FEEDBACK_RELAY"); if relay == "" { relay = "https://feedback.intrane.fr" }
    relayed := false
    if relay != "off" {
        client := &http.Client{Timeout: 5 * time.Second}
        if resp, err := client.Post(relay+"/v1/feedback", "application/json", bytes.NewReader(payload)); err == nil {
            relayed = resp.StatusCode == 200; resp.Body.Close()
        }
    }
    out, _ := json.Marshal(map[string]any{"ok": true, "id": id, "relayed": relayed})
    fmt.Println(string(out))
}
```

**Shell** clients dual-write with two `curl` calls carrying the same `id`; see `grepapi`.

## Step 4 — wire the subcommand + document it

- Add `feedback` to your CLI's command dispatch (before it opens a DB, so it works without
  local state).
- Add a short **Feedback** section to your README naming the command, `FEEDBACK_RELAY`, and
  your app's URL env var. (Copy one from a reference tool.)
- Bump your version.

## Verify

```sh
# endpoint stores
curl -s -XPOST http://localhost:PORT/v1/feedback -H 'content-type: application/json' \
  -d '{"message":"hello","kind":"idea","id":"t1"}'          # -> {"ok":true,"id":"t1","stored":true}
curl -s -XPOST http://localhost:PORT/v1/feedback -d '{"kind":"bug"}'   # -> 400 message required

# dual-write (relay off during local test)
MYAPP_URL=http://localhost:PORT FEEDBACK_RELAY=off myapp feedback "cli test" -kind bug
# -> {"ok":true,"data":{"id":"…","stored":1,"relayed":0}}

# then live, and confirm the relay recorded your app:
curl -s "$RELAY/v1/feedback?app=myapp" -H "authorization: Bearer $FEEDBACK_ADMIN_TOKEN"
```

## Gotchas seen in the field

- **Escape the message.** Build the outbound JSON with your language's JSON encoder (machin
  `json(x)`), never string concatenation of raw user text.
- **Watch name collisions.** `client_ip`, `route`, etc. may already be defined by your web
  framework — prefix yours if so.
- **The relay stores nothing if it can't write its file.** If you self-host the relay, make
  sure its data directory is writable by the daemon user, or it silently 200s without
  persisting.
- **Dispatch `feedback` before any DB open** in the CLI, so a relay-only submission works on
  a machine with no local database.
