# feedback-spec — a feedback command convention for agent-first CLIs

A tiny, copy-pasteable convention for giving any command-line tool a **`feedback`**
command that goes two places at once:

1. the tool's **own** feedback endpoint (so each app owns its feedback), and
2. a **central relay** (so one inbox spans every tool you ship).

It is deliberately small — a JSON body, one `POST` route, one CLI subcommand — so an agent
can add it to a new tool in a few minutes, and so tools written in different languages all
land in the same place.

```sh
myapp feedback "the reconnect is flaky on wifi handoff" -kind bug -context "long-running tunnel"
# -> {"ok":true,"data":{"id":"…","stored":1,"relayed":1}}
```

## Why

Agent-operated tools have no "report a problem" button and no human watching a dashboard.
Feedback — from the human *or* from the agent itself ("this flag is confusing", "this
errored") — has nowhere to go. This convention gives it somewhere to go with almost no
code, and makes cross-tool feedback aggregatable without coupling any tool to a specific
backend.

## The three documents

| File | For | What it is |
|---|---|---|
| **[PROTOCOL.md](PROTOCOL.md)** | implementers | the normative contract (MUST/SHOULD): wire format, endpoint, CLI, storage, guarantees |
| **[RECIPE.md](RECIPE.md)** | implementers | the 4-step how-to with reference snippets (machin + Go) |
| **[llms.txt](llms.txt)** | agents | the machine-first entry point — fetch this first |
| **[feedback.schema.json](feedback.schema.json)** | everyone | JSON Schema for the submission body |
| **[SKILL.md](SKILL.md)** | agents | drop-in skill: "add a feedback command to this CLI" |

## The contract in one breath

- **Submit** is `POST /v1/feedback` with a JSON body `{app, version, kind, message, context, reporter, id}`. Open (no auth), **rate-limited**, **size-capped**, **idempotent on `id`**.
- The **CLI** `feedback` subcommand **dual-writes**: to the app's own endpoint and, best-effort, to the relay. It **never fails the caller** — it reports `stored` / `relayed` and exits 0.
- **Reading** feedback is **admin-gated** (Bearer token). Submission is open by design.
- The relay is **swappable** (`FEEDBACK_RELAY`, or `off`). Nothing forces you onto a specific backend.

Full details in [PROTOCOL.md](PROTOCOL.md).

## Reference implementations

The reference **relay** is [machin-feedback](https://github.com/javimosch/machin-feedback)
(live at `https://feedback.intrane.fr`). These tools implement the client side across
several languages and architectures — each is a worked example:

| Tool | Language | Adoption shape |
|---|---|---|
| [hart](https://github.com/javimosch/machin-hart) | machin | native client, dual-write |
| [grepapi](https://github.com/javimosch/grepapi) | machin + shell | shell client, dual-write |
| [crmd](https://github.com/javimosch/crmd) + [crm-cli](https://github.com/javimosch/crm-cli) | machin | server endpoint + separate client |
| [remotecmd](https://github.com/javimosch/remotecmd) | Go | relay-only (peer tool, no local store) |
| [portier](https://github.com/javimosch/portier) | machin | endpoint + CLI |
| [machin-idp](https://github.com/javimosch/machin-idp) | machin | endpoint + CLI |
| [chatsnip](https://github.com/javimosch/chatsnip) | machin | endpoint + CLI |

That the same contract fit a native client, a shell client, a server-plus-client split, and
a relay-only peer without changing shape is the reason it's written down here.

## License

MIT — see [LICENSE](LICENSE). This is a convention; copy it, fork it, run your own relay.
