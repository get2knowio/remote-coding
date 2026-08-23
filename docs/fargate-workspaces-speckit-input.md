# Speckit Specify Input: Fargate Ephemeral Workspaces (`remo fargate`)

Use the contents of this document as the feature description for Speckit's
`specify` prompt. Preserve the decisions marked **Required decision** when
generating the specification and the later implementation plan. Items marked
**Open question** are deliberately unresolved and should surface as
clarifications during `specify`/`clarify`, not be silently decided.

Provenance: distilled from a design investigation (2026-08-23) into whether
remo could run each devcontainer as its own ECS Fargate task, with `remo
shell` and the web console attaching to tasks on demand and idle tasks
scaling to zero.

## Feature Summary

Add a new `fargate` provider in which **one ECS Fargate task is one
devcontainer is one session** — there is no host. Where every existing remo
provider provisions a machine that hosts N projects (each a devcontainer with
a zellij session, reached through `remo-host`), a Fargate workspace *is* the
project: `remo fargate create` registers a per-project identity and an image
build; `remo shell <name>` resolves the project's running task (starting one
if none is running), and connects into the devcontainer directly. Idle
workspaces stop themselves and cost nothing while stopped; durable state
lives on EFS so a restarted workspace resumes where it left off — including
the coding agent's conversation (`claude --continue`).

The web console becomes a first-class creation surface for this provider:
"New workspace" launches a task and opens a browser terminal, with none of
the trust-bootstrap ceremony that attaching to foreign machines requires,
because the service created the compute itself.

This is remo's **ephemeral tier**, sitting alongside the existing providers
(Incus, Proxmox, Hetzner, AWS EC2), not replacing them. Users who want a real
disk, instant attach, and multi-project density keep using those. The
positioning is deliberate: **terminal-first, agent-aware, your-account** — not
an IDE workbench. Codespaces and Coder are IDE-first products with heavy or
vendor-owned control planes; the lane this feature occupies (persistent
terminal sessions, coding-agent-aware lifecycle, single operator,
multi-provider alongside a home lab) is the lane they structurally leave
open. The spec must not accrete IDE-workbench features (web editors,
extension marketplaces, collab editing, org SSO).

## Motivation

remo's current model assumes a long-lived host you pay for (or power) whether
or not you are using it. The EC2 provider mitigates this with stop/start, but
a stopped VM still needs manual lifecycle discipline and still carries the
maintenance burden of a mutable, Ansible-configured machine. Fargate offers
per-second billing, zero host maintenance, and genuine scale-to-zero — an
excellent fit for the dominant remo workload: intermittent, agent-assisted
coding sessions where the machine sits idle most of the day.

Separately, the web console's hardest-won machinery (features 011/012/017/
022/023: pairing, deploy-key trust bootstrap, fingerprint confirmation,
three-way registry sync, flap detection) all exists to answer one question —
*how does the console safely attach to compute whose creation it never
witnessed?* For compute the service creates, those problems dissolve rather
than get solved: creation is trust, and ECS is the single authoritative
store, so there is nothing to three-way merge.

## Terms and Definitions

| Term | Definition |
|---|---|
| **Workspace** | The durable per-project identity: a task-definition family plus its EFS access point and ECR image. Exists whether or not a task is running. |
| **Task** | One running ECS Fargate task for a workspace. At most one per workspace. Ephemeral: memory and task storage die with it. |
| **Task-definition family** | The stable ECS name for a workspace's task definition revisions. The registry stores this, never a task ARN. |
| **ECS Exec managed agent** | The SSM agent Fargate (platform 1.4) injects into every container when `enableExecuteCommand` is set. Registers the container as SSM target `ecs:<cluster>_<task-id>_<runtime-id>` with zero in-image setup. |
| **sshd-over-SSM** | The connection strategy: a small sshd in the workspace image, reached through an SSM session to the ECS Exec target — no inbound ports, no public IP, host identity verified by IAM rather than host keys. |
| **envbuilder** | Coder's OSS tool that builds a devcontainer *inside* an unprivileged container at startup (kaniko-style), with registry layer caching. The candidate answer to "Fargate cannot run `devcontainer up`". |
| **Prebuild pipeline** | The alternative image strategy: `devcontainer build` runs in CodeBuild/GitHub Actions, pushes to ECR, tasks run the finished image. |
| **Idle watchdog** | The in-task supervisor that stops the task by exiting when the workspace is idle. |
| **Agent idleness** | The state where the coding agent has finished its turn and is waiting for human input, as reported by Claude Code's own `Stop`/`Notification` hooks — distinct from "no terminal attached" and from "low CPU". |
| **Reaper** | The external EventBridge-scheduled Lambda backstop that stops remo-tagged tasks past a hard ceiling or with a dead watchdog. |
| **Pseudo-suspend** | Stop/start with all durable state (workspace, `$HOME`, `~/.claude` transcripts) on EFS. Memory is always lost; the agent conversation is not. |

## Required Architectural Decisions

### 1. Model: workspace = task, no host

**Required decision.** One task per workspace, one devcontainer per task.
The host/session two-level hierarchy collapses; `remo-host`'s session list
for a Fargate workspace is degenerate (exactly one target). A terminal
multiplexer (zellij) still runs *inside* the task so terminal state survives
client disconnects while the task lives.

### 2. Connection: sshd-over-SSM, riding the existing proxy seam

**Required decision.** The workspace image runs sshd; the CLI and web service
reach it through SSM against the ECS Exec target ID. This slots into the
existing `ConnectionSpec.proxy_hook` → `SshProxyPlan` seam (the AWS EC2 SSM
ProxyCommand in `providers/aws.py` is the worked example) and therefore
preserves — unchanged — `remo cp` (rsync), `-L` tunnels, the pre-connect
version check, and the web console's `ssh_master`/`terminal.py` path.
`StrictHostKeyChecking=no` is correct here for the same reason it is for
EC2-over-SSM: target identity is IAM-verified.

Native ECS Exec (`aws ecs execute-command`, no sshd) is **rejected** as the
primary path: it is not SSH, so it forfeits rsync, tunnels, and the entire
web terminal stack, and would force a new connection variant into `core/`.

### 3. Lifecycle: standalone RunTask, never an ECS service

**Required decision.** Workspaces are launched with `RunTask` against the
task-definition family — not as ECS services. Two reasons: a service's
`desiredCount=1` fights the idle watchdog (ECS would restart what the
watchdog just stopped), and standalone tasks let the watchdog stop the task
by simply **exiting PID 1** — the essential container exiting stops the task
and the billing, with zero AWS permissions needed inside the task.

`remo shell <name>` with no running task performs RunTask, waits for RUNNING
(cold start ~30–90s, image-pull dominated), then connects — the same UX as
today's `auto_start_aws_if_stopped`.

### 4. Image build: devcontainer fidelity without a Docker daemon

Fargate permits no privileged containers, no Docker daemon, no DinD —
`devcontainer up` cannot run in a task, period. The spec must preserve
devcontainer.json fidelity (features, image, lifecycle hooks); a generic
"universal image" that ignores the project's devcontainer definition is
**rejected**.

**Open question (primary fork).** envbuilder (build inside the unprivileged
task at start, ECR layer cache, slow first boot, no pipeline) versus a
prebuild pipeline (CodeBuild/Actions → ECR, fast boots, a pipeline
round-trip per devcontainer.json change). Leaning envbuilder for v1
(single-operator, no extra infra), but the spec should treat this as its
first clarification.

Either way, the image bakes: sshd, zellij, a slim `remo-host` (degenerate
sessions list), the Claude Code idle hooks, and `~/.remo-version` (written at
build — the Ansible layer is **bypassed entirely** for this provider;
`upgrade` means rebuild + redeploy).

### 5. State: EFS pseudo-suspend; suspension does not exist

**Required decision.** Fargate tasks cannot be paused, checkpointed, or
snapshotted (no CRIU, no exposed Firecracker snapshots). "Suspend" is
simulated: an EFS access point per workspace holds the project tree and
`$HOME` (including `~/.claude`), so a restarted task resumes the agent
conversation via `claude --continue`. Memory and running processes are
always lost on stop — the spec's UX copy must say "stop", never "suspend".

Known cost to document honestly: EFS latency on `node_modules`/git-heavy
trees. **Open question:** whether v1 also offers a cattle mode (no EFS; the
git remote is the state, with an aggressive unpushed-work stop-guard).

Snapshot verbs raise `PreconditionError` (unsupported) for this provider.

### 6. Idle detection and scale-to-zero

**Required decision.** Idleness is agent-aware, not merely terminal-aware.
Low CPU and "no client attached" are both wrong alone: an autonomous agent
mid-task (`remo shell --detach --exec 'claude …'`) has no attached human and
must never be reaped. The baked Claude Code hooks publish agent state —
`Stop`/`Notification` write "waiting since T"; tool-use hooks write
"working" — and the watchdog's rule is: stop when the agent has been
*waiting* for N minutes AND no client is attached (`who`/established sshd
sessions).

Safety layers, both required:
- **Stop-guard**: before exiting, check `git status`/unpushed commits (the
  same signals `remo-host sessions list` computes today). Dirty → auto-push
  to a `wip/` branch or defer and extend the timer. Use Fargate's
  `stopTimeout` (up to 120s after SIGTERM) to flush.
- **Reaper backstop**: an EventBridge-scheduled Lambda stops remo-tagged
  tasks past a hard max age or whose watchdog has died. In-task logic
  handles the smart cases; the reaper handles the wedged ones.

### 7. Identity, registry, and discovery

**Required decision.** The registry stores the durable identity only
(workspace name, cluster, task-definition family, region); the task ARN is
resolved live at connect time. ECS tags are the managed marker
(`supports_managed_marker=True`, native tagging, no backfill). Discovery of
workspaces created elsewhere (e.g. by the web console) rides the existing
016 sync-reconcile machinery — but note the inversion: for this provider ECS
is the authoritative store both planes read, so registry entries are cache,
and the 023 three-way-merge problem does not arise for `type="fargate"`
entries.

### 8. Web console: creation surface, not attachment surface

For Fargate workspaces the console **creates** compute rather than attaching
to foreign machines, so the scan-key/trust-key/verify wizard does not apply —
creation is trust. "New workspace" runs the embedded CLI (`remo fargate
create`) as a detached web job (the 023 `web/jobs.py` pattern, unchanged);
the rail shows stopped workspaces with start affordances and running tasks
with a burn indicator. The console itself may run as a small always-on
Fargate service (task-role credentials — preferred) or remain a home-lab
container with mounted AWS credentials (works; weaker posture). Nothing in
the system requires an inbound port: tasks have no public IPs and no
ingress; all reachability is SSM egress.

### 9. Security posture (day one, not hardening later)

The console becomes a **compute-minting surface**: a compromised session can
create billed infrastructure, not just attach to existing machines. Required
in the spec from the start: operator auth is load-bearing (not
belt-and-braces); the service's IAM role is scoped to remo-tagged
task-definition families with bounded cpu/memory; per-operator task quotas;
a budget alarm in the reference deployment.

### 10. Constitution fit and the named core seams

Principle II holds: `fargate` is one implementation module plus one
descriptor. The two known places where core must grow a seam (by PR, as
descriptor fields — never `type` literals in `core/`):
- **Pre-connect hook**: `cli/shell.py`'s `auto_start_aws_if_stopped` is
  already an AWS literal in the CLI layer; Fargate's start-on-connect is the
  same shape. Generalize into a descriptor hook and migrate AWS onto it.
- **Provider-declared unsupported verbs** (snapshots) if the current
  Protocol surface cannot already express refusal cleanly.

## Sketch User Scenarios

1. `remo fargate create myapp --repo git@github.com:me/myapp` builds (or
   registers a build for) the workspace image, creates the EFS access point
   and task-definition family, and registers the workspace. No task runs yet.
2. `remo shell myapp` finds no running task, starts one, waits for RUNNING,
   and drops into the devcontainer's zellij session. `remo cp` and `-L`
   work as on any other provider.
3. The user asks Claude Code a question, gets an answer, and walks away. The
   Stop hook marks the agent waiting; after N quiet minutes with no client
   attached, the watchdog verifies the tree is clean (or pushes `wip/`),
   and exits. The task stops; cost drops to EFS + ECR storage.
4. Next morning, `remo shell myapp` cold-starts a fresh task; `claude
   --continue` resumes yesterday's conversation from EFS.
5. `remo shell myapp --detach --exec 'claude …'` leaves an agent working
   with no human attached; the watchdog does not reap it while tool-use
   hooks report activity. The reaper's hard ceiling still applies.
6. In the web console, "New workspace" creates and starts a workspace and
   opens a browser terminal — no fingerprint ceremony. The workspace later
   appears on the workstation via `remo fargate sync`.

## Hosted Mode (Future Direction — Constrain Now, Build Later)

A plausible commercial future for this feature is a **hosted control plane
with BYO compute** — the Tailscale model. Tasks always run in the user's own
AWS account (via a scoped cross-account role); what is hosted is the console:
zero-setup reachability (no Tailscale/reverse-proxy/TLS/forward-auth
homework), real OIDC auth, multi-device pairing, hosted reaper and budget
guardrails, and — the marquee capability — **mobile push when the agent needs
you**, driven by the same Claude Code `Stop`/`Notification` hooks that drive
the idle watchdog. Self-hosted `remo web serve` remains the free,
fully-capable Headscale-equivalent forever; the paid tier sells hosting and
convenience, never gated features.

**Explicitly rejected**: remo hosting the compute itself (subscription
covering compute plus margin). That model inherits compute-resale margins,
mining abuse, tenant isolation, and compliance burden while discarding the
"your account, your infra" differentiation.

The v1 spec does **not** build any of this — no billing, no multi-tenant
service, no mobile app. But two constraints are cheap to honor now and
expensive to retrofit, so the v1 design must not foreclose them:

1. **Accelerator, never a runtime dependency.** The CLI holds the user's AWS
   credentials and talks to ECS/SSM directly; every workflow in this spec
   must work with no console reachable at all. A future hosted plane makes
   things nicer, never possible. (The ECS-is-the-store property in Decision 7
   is what lets CLI and any console converge without a sync protocol —
   preserve it.)
2. **Keep the terminal data plane separable from the console.** SSM's
   session protocol is a documented websocket to
   `ssmmessages.<region>.amazonaws.com`; a browser holding short-lived
   session tokens can carry terminal bytes **directly to AWS**, so a hosted
   control plane need never see session content (a security and privacy
   property worth having). v1 may keep today's server-side PTY path, but the
   connection design should not bake in an assumption that the console
   process must sit on the terminal byte path.

Existing seams that a hosted mode would grow through, and which v1 should
therefore not erode: `deployment_id`, the pluggable `operator_auth` provider
(OIDC deferred), and the hook-driven agent-state file (which doubles as the
notification event source).

## Out of Scope (v1)

Multi-user/multi-tenant anything; Fargate Spot; GPU tasks; VS Code Web /
code-server bundling; Windows containers; cold-start optimization beyond
image-size discipline (SOCI lazy loading is a noted follow-up); provider
parity for `resize` beyond cpu/memory task-definition revisions; migrating
existing provider hosts into Fargate workspaces. Also out of scope: the
hosted control plane itself, billing, mobile push delivery, and
browser-direct SSM transport — v1 only preserves the seams above.

## Open Questions (surface during clarify)

1. envbuilder vs prebuild pipeline (see Decision 4) — the primary fork.
2. EFS-always vs optional cattle mode (Decision 5).
3. Idle threshold defaults and whether the stop-guard auto-pushes or defers.
4. Whether `remo fargate` v1 includes the web "New workspace" surface or
   ships CLI-first with the console read-only for this type.
5. Cluster/VPC provisioning: does remo create the cluster, subnets, and EFS
   on first use (a bootstrap verb, like `incus bootstrap`), or require
   pre-existing infrastructure named in config?
6. How `remo web` deployments *not* running in AWS obtain credentials for
   this provider, and whether that path is supported or documented-against.
