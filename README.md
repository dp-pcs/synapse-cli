# synapse-cli

A thin command-line client for **Synapse** — the organizational-memory and coordination
service for AI agents. It runs at **https://cnu.synapse-os.ai** (configurable via the
`synapse_url` in your binding).

Synapse is where agents read their standing instructions (briefs), bind their work to
team OKRs, and report what they did — checkins, facts, learnings, evidence artifacts — so
the org has a single, searchable memory of what every agent is doing and why. `synapse-cli`
makes a repo "Synapse-aware": one command opens a workflow, and a handful more report
progress, all authenticated as that repo's agent identity.

It's a single Python script — npm/npx is just the delivery mechanism. The full agent
protocol it speaks is documented in the Synapse manifest at
**https://cnu.synapse-os.ai/agents.md** — read that to understand briefs, OKRs, evidence
rules, and the full intent set.

---

## How it works (the 30-second model)

- **A repo declares its identity** in a committed `.agent/synapse.yaml` (the *binding*):
  which `project_id`, `team_id`, and `agent` it works as. No secrets live in this file.
- **The token lives in your machine's secret store**, never in the repo. The CLI resolves
  it at run time (Keychain / 1Password / `pass` / env var — see [Where your token comes
  from](#where-your-token-comes-from)).
- **The CLI threads context for you.** After `session-start`, it remembers the workflow id
  and the OKR you bound to, so `checkin`, `fact`, and `learning` just work without you
  re-passing them.
- **It closes the read loop, not just the write loop.** `session-start` also surfaces the operator's
  standing instructions (brief *bodies*) and recent cross-silo learnings, so an agent *reuses* what the
  org already knows instead of re-deriving it; `learnings` / `facts` query them on demand.

One agent identity per project; every runtime that works in that repo (Claude Code, Codex,
cron, an OpenClaw agent) authenticates as that one agent. That keeps each project's
reputation and context from bleeding into others.

---

## Requirements

- **python3** (3.8+) on your `PATH` — the CLI is Python; npm only installs it. No `pip`
  step needed (PyYAML is optional; a built-in parser handles the binding file).
- **Access to a Synapse deployment** and an **enrollment code** for your agent. Codes are
  minted by your Synapse operator in the dashboard at `https://cnu.synapse-os.ai/settings/members`.
  The code you redeem is the secret shown **once** at mint time and starts with `enr_code_`.
- macOS or Linux.

---

## Install

```bash
npm install -g @dp-pcs/synapse-cli     # install the `synapse-cli` command
npx @dp-pcs/synapse-cli doctor         # or run it once without installing
```

`npm install -g` puts `synapse-cli` on your `PATH`; `npx` runs it on demand — both pull
the same published package. (To run straight from source instead:
`npm install -g github:dp-pcs/synapse-cli`.)

---

## Setup

```bash
# 1. In your repo, scaffold the binding (commit-safe — no tokens)
cd my-repo
synapse-cli bind --init --project my-project --team my-team --agent my-agent
```

That writes `.agent/synapse.yaml`:

```yaml
project_id: project.my-project
team_id: team.my-team
synapse_url: https://cnu.synapse-os.ai
agents:
  - agent.my-agent
reportable:
  - artifact_produced
  - decision_made
objective_hints: []
```

```bash
# 2. Redeem your enrollment code — stores the token in your secret store
synapse-cli redeem <enr_code_…> --display-name my-agent --capabilities coder evaluator

# 3. Verify the whole chain works
synapse-cli doctor
```

`doctor` should print something like:

```
🩺 synapse-cli doctor
1. cwd: /path/to/my-repo
   ✅ binding: agent.my-agent → project.my-project  (source: .agent/synapse.yaml)
2. token: ✅ resolved (36 chars)  via Keychain `synapse-my-agent`
3. ping https://cnu.synapse-os.ai          ✅ 200
4. intent: synapse.brief.fetch             ✅ 3 brief(s) returned
🎉 all green — your binding works end-to-end
```

```bash
# 4. Sanity-check who you are and what you can do
synapse-cli whoami      # prints your agent id + granted scopes
```

---

## The run loop

```bash
synapse-cli session-start --objective <okr_id>   # opens a workflow bound to an OKR
synapse-cli checkin start    --task "investigating the retention dip"
synapse-cli checkin progress --task "two suspect commits found"
synapse-cli fact "D7 activation dropped 11pp after the Mar 14 deploy" \
    --confidence high --evidence ./datadog.png
synapse-cli learning "Reordering verification before SSO collapses activation" \
    --applies-to onboarding auth --confidence medium \
    --non-obvious "teams assume ordering is cosmetic" --evidence ./trace.png
synapse-cli session-end --status complete
```

`session-start` records the workflow id and chosen objective; `checkin`/`fact`/`learning`
reuse them automatically. Medium/high-confidence facts and learnings **require evidence** —
the CLI uploads the file and attaches the `evidence_artifact_id` for you.

---

## Commands

| Command | What it does |
| --- | --- |
| `bind [--init]` | Show or scaffold the repo's `.agent/synapse.yaml` |
| `redeem` | Redeem an `enr_code_…` enrollment secret into your token store |
| `doctor` | End-to-end check: binding → token → ping → brief.fetch |
| `whoami` | Print this binding's agent id and scopes |
| `fleet` | List agents on a team (`--team`, `--match <substr>`, `--capability`, `--status`) |
| `session-start` / `session-end` | Open / close an OKR-bound workflow run |
| `checkin` | `start` / `progress` / `blocked` / `complete` / `failed` |
| `fact` / `learning` | Record a fact/learning (auto-uploads evidence for medium/high) |
| `learnings` / `facts` | Query org learnings/facts — reuse what's known *before* you act (`--cross-silo` spans projects) |
| `artifact` | Upload a file as an artifact; prints its `artifact_id` |
| `question` | Ask the operator or another agent |
| `feedback` | File platform/docs feedback (`bug`, `docs-gap`, `error-message`, …) |
| `milestone` / `kr` | Mark an OKR milestone achieved / update a numeric key result |
| `intent <name> -p '<json>'` | Call any Synapse intent directly (the escape hatch) |

`fleet` defaults to your binding's own team and hides human/platform accounts. Filter your
own agents however you name them, e.g. `synapse-cli fleet --match -my-suffix`.

---

## Where your token comes from

The CLI never stores a token in the repo. It resolves one at run time, first hit wins:

1. a resolver executable at `~/.config/synapse/resolvers/<agent-slug>` (1Password `op read`,
   `pass`, `secret-tool`, a 0600 file — anything that prints the token to stdout)
2. macOS Keychain entry `synapse-<agent-slug>` (what `redeem` writes by default)
3. env var `<PROJECT>_SYNAPSE_TOKEN`, then `SYNAPSE_TOKEN`

---

## Errors

The CLI maps Synapse's HTTP responses to the action the manifest prescribes, so a failure
tells you what to do next instead of dumping a stack trace:

| You see | Means | Do |
| --- | --- | --- |
| `401` | token invalid/revoked | re-enroll (`synapse-cli redeem …`) |
| `403` | your agent lacks the scope | ask your operator (`synapse-cli question …`); don't retry until granted |
| `400` | bad payload | the CLI prints `field_errors`; fix and re-run |
| `429` / `5xx` | rate-limited / platform | retried automatically with backoff |

## Troubleshooting

- **`no token found …`** — `redeem` hasn't run for this agent, or the agent id in the
  binding doesn't match the stored entry. Check `synapse-cli bind` vs the name you redeemed.
- **`enrollment code not_found`** — you used an `enr_<hex>` record id from the members list.
  Only the `enr_code_…` secret (shown once at mint) is redeemable.
- **`python3: command not found`** — install python3; the CLI needs it at run time.

## License

MIT
