# synapse-cli

A thin client for the **Synapse** organizational-memory API. It binds a repo to an
agent identity, opens OKR-bound workflow runs, and reports work — checkins, facts,
learnings, artifacts — back to Synapse so your agents' activity logs itself.

It's a single Python script. npm/npx is just the delivery mechanism.

## Requirements

- **python3** (3.8+) on `PATH` — the CLI is Python; npm only installs it.
- A token store for your agent token. Resolution order (first hit wins):
  1. a resolver executable at `~/.config/synapse/resolvers/<agent-slug>` (1Password, `pass`, secret-tool, a file — anything that prints the token to stdout)
  2. macOS Keychain entry `synapse-<agent-slug>`
  3. env var `<PROJECT>_SYNAPSE_TOKEN` or `SYNAPSE_TOKEN`
- No PyYAML required — bindings are parsed with a built-in fallback parser (PyYAML is used automatically if present).

## Install

You don't need to publish to the npm registry — install straight from GitHub:

```bash
# run once without installing
npx github:dp-pcs/synapse-cli doctor

# or install the command globally
npm install -g github:dp-pcs/synapse-cli
synapse-cli doctor
```

`npx` runs it on demand; `npm install -g` puts `synapse-cli` on your `PATH` permanently. Same tool.

## Quickstart

```bash
# 1. Scaffold a binding in your repo (commit-safe — no tokens stored here)
cd my-repo
synapse-cli bind --init --project my-project --team my-team --agent my-agent

# 2. Mint the agent's token (redeem the enr_code_… secret shown once at mint time)
synapse-cli redeem <enr_code_…> --display-name my-agent --capabilities coder evaluator

# 3. Verify end to end
synapse-cli doctor

# 4. Confirm who you are and what scopes you hold
synapse-cli whoami
```

## The run loop

```bash
synapse-cli session-start --objective <okr_id>   # binds the run to an OKR (auto-picks if the project has exactly one active OKR)
synapse-cli checkin start  --task "investigating the retention dip"
synapse-cli checkin progress --task "found two suspect commits"
synapse-cli fact "D7 activation dropped 11pp after the Mar 14 deploy" --confidence high --evidence ./datadog.png
synapse-cli learning "Reordering verification before SSO collapses activation" --applies-to onboarding auth --confidence medium --non-obvious "teams assume ordering is cosmetic" --evidence ./trace.png
synapse-cli session-end --status complete
```

`session-start` records the workflow id and chosen objective; `checkin`, `fact`, and
`learning` reuse them automatically. Medium/high-confidence facts and learnings require
evidence — the CLI uploads the file and attaches the `evidence_artifact_id` for you.

## Commands

| Command | What it does |
| --- | --- |
| `bind [--init]` | Show or scaffold the repo's `.agent/synapse.yaml` binding |
| `redeem` | Redeem an `enr_code_…` enrollment secret into your token store |
| `doctor` | End-to-end check: binding → token → ping → brief.fetch |
| `whoami` | Print this binding's agent id and scopes |
| `fleet` | List agents on a team (`--team`, `--match <substr>`, `--capability`, `--status`) |
| `session-start` / `session-end` | Open / close an OKR-bound workflow run |
| `checkin` | Check in: `start` / `progress` / `blocked` / `complete` / `failed` |
| `fact` / `learning` | Record a fact/learning (auto-uploads evidence for medium/high) |
| `artifact` | Upload a file as an artifact; prints its `artifact_id` |
| `question` | Ask the operator or another agent |
| `milestone` / `kr` | Mark an OKR milestone achieved / update a numeric key result |
| `intent <name> -p '<json>'` | Call any intent directly (the escape hatch) |

`fleet` is generic — it defaults to your binding's own team and hides platform/human
accounts. Filter your own agents however you name them, e.g. `synapse-cli fleet --match -my-suffix`.

## Errors

The CLI maps Synapse's HTTP responses to the runbook's prescribed actions: `401` → re-enroll,
`403` → scope missing (ask your operator), `400` → shows `field_errors`, `429`/`5xx` → retried
with backoff. A failure tells you what to do next rather than dumping a stack trace.

## License

MIT
