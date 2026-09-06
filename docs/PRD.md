# Nook — Product Requirements Document

> **A cloud for small software.** Deploy and share bespoke, agent-built internal tools as easily as sharing a Google Doc.

- **Status:** Draft v0.1
- **Last updated:** 2026-09-06
- **Owner:** Shrikant Shingare

---

## 0. How to read this document

Every item below is tagged:

- **[LOCKED]** — a decision explicitly made and approved. Do not change without a new decision.
- **[PROPOSED]** — my recommendation, awaiting your sign-off. Nothing here is final.
- **[OPEN]** — an unresolved question captured for a future decision.

I did not decide anything on my own. All **[LOCKED]** items came from your answers; everything else is flagged for you.

---

## 1. Summary

**Nook** is a cloud purpose-built for *small software* — purpose-built tools that will only ever have one or a small handful of users. Agents (Claude Code, v0, etc.) have made such tools cheap to *build*; Nook makes them trivial to *deploy and share*.

Incumbent clouds (AWS, Azure) were designed for big software that scales to many users at the cost of enormous complexity. Nook deletes that complexity for the small-software case: a builder points an agent (or drags a folder) at Nook, and gets back a shareable URL that their colleagues — technical or not — can open, log into, and use.

Under the hood, every app runs in its own **Firecracker microVM** (hardware-strength isolation), so non-technical people can safely share arbitrary, agent-generated code across a company.

---

## 2. Background & motivation

Distilled from the "A Cloud for Small Software" thesis:

- **Small software is now easy to build, but still hard to deploy and share.** The bottleneck moved from creation to distribution.
- **There is unlimited demand for bespoke team tools** — running workflows, tracking important numbers, managing sprints, sharing prototypes. Every team does things differently.
- **The incumbent clouds are the wrong shape.** They optimize for scale-to-many-users and pay for it in complexity. Small software never needs that scale, so most of the complexity can be deleted.
- **The hard problems that remain:** (a) every company wants to customize the environment the software runs in; (b) auth and permissions; (c) letting *non-technical* users share arbitrary code *securely*.
- **The north star:** small software should be as easy to share with colleagues as a Google Doc.

---

## 3. Goals & non-goals

### 3.1 Goals

1. **Delete deployment complexity** for small software: source → live URL in one step, zero infra config.
2. **Make sharing Google-Doc-simple**: a link + org-scoped permissions.
3. **Run arbitrary, untrusted, agent-generated code safely** via per-app microVM isolation.
4. **Serve both technical builders and non-technical colleagues** from one product.
5. **Support environment customization** for the apps that need it (Tier 2).
6. **Be agent-native**: the primary deploy path is "the agent deploys for you."

### 3.2 Non-goals (for now)

- Not a platform for **big, high-scale, many-user software** — that's AWS/Azure's job.
- Not a **general-purpose Kubernetes/PaaS** — no cluster management surfaced to users.
- Not an **IDE / code-generation product** — Nook deploys and shares; agents build.
- Not **on-prem / bring-your-own-cluster** in v1 (see §14 Future).

---

## 4. Target audience & personas — [LOCKED]

**Primary market:** small teams inside companies (startups first, departments of larger companies later).

| Persona | Who | Technical level | Need |
|---|---|---|---|
| **The Builder** (wedge) | Founder, PM, ops lead, or a single engineer on a team | Semi-technical → technical; already builds with agents | "I built a tool with my agent — now get it to my teammates without fighting a cloud." |
| **The Colleague** (end-state) | Anyone the builder shares with | Often **non-technical** | "Open a link, log in, use the tool." |
| **The Admin** (later) | Team/org owner | Varies | "Control who can deploy and who can access what." |

**Adoption motion:** bottom-up, self-serve. The Builder brings Nook into the org; usage spreads to Colleagues; Admin/governance features monetize and retain.

---

## 5. Business model — [LOCKED]

**Open-core SaaS (YC-shaped).**

- **Hosted, multi-tenant SaaS** is the business — this is what a non-technical, Google-Doc-style audience requires (they will never self-host).
- **Open-source wedge**: open-source the **CLI, SDK, and runtime conventions** to win developer trust and drive bottom-up adoption.
- **Monetize the hosted platform** + team/enterprise features: SSO, permissions/governance, audit logs, custom environments, higher limits, private networking.

**[PROPOSED] Pricing shape** (needs your sign-off — see §13):
- **Free**: a few apps, personal use, Nook subdomain.
- **Team** (per-seat, per-month): unlimited small apps, org SSO, permissions, custom subdomain.
- **Enterprise**: SAML, audit, environment customization, SLAs.

---

## 6. Product overview — the two tiers — [LOCKED]

Both tiers **collapse onto the same runtime**: "a root filesystem booted in a Firecracker microVM." They differ only in *who builds that filesystem*.

| | **Tier 1 — Source deploy (default)** | **Tier 2 — Bring-your-own-container (advanced)** |
|---|---|---|
| Who it's for | Everyone (90% case) | Technical builders needing a custom environment |
| Input | App source (files/repo) | Dockerfile-from-source **or** prebuilt image |
| Stack | Auto-detected (TS/JS, Python) | User-defined via container |
| Rootfs built by | **Nook** (from detected source) | **User** (their image) → Nook flattens to rootfs |
| Visibility | Default, front-and-center | Behind an "advanced" toggle |

> **Design principle:** the user never categorizes their app as "frontend" or "backend." They ship *one app*; Nook figures out what's static, what's a server endpoint, and wires them behind one URL.

---

## 7. Detailed requirements

### 7.1 Tier 1 — source deploy — [LOCKED]

- **Unit of deploy:** a project folder / repo = one app.
- **Auto-detection** of app shape:
  - *Static* (raw HTML/CSS/JS or a built SPA) → served from edge/CDN.
  - *Full-stack JS/TS* (Next.js, Vite + `/api`, or a single server file) → static to CDN, server to microVM.
  - *Backend API* (Python/Node service) → microVM; optional static frontend attached.
- **Supported runtimes (v1):** **TypeScript/JavaScript** and **Python**. Anything else → route to Tier 2.
- **Detection signals:** `package.json` (Node/JS; inspect deps/scripts), `requirements.txt` / `pyproject.toml` (Python), root `index.html` with no build (static).
- **Override manifest** `smallapp.json` (optional, agent-writable) removes all ambiguity:
  ```json
  {
    "runtime": "node",
    "build": "npm run build",
    "start": "npm run start",
    "routes": { "/api": "server", "/": "static" }
  }
  ```
- **Output:** a live shareable URL (see §7.5) with permissions applied.

### 7.2 Tier 2 — bring-your-own-container — [LOCKED]

- **Accepts both** (a) a **Dockerfile-from-source** (Nook builds it) and (b) a **prebuilt image** from a registry (Nook pulls it).
- **Build is treated as untrusted** — Dockerfile `RUN` steps execute arbitrary code, so builds run in a **sandboxed builder** (rootless BuildKit or an ephemeral microVM), never on a shared privileged daemon.
- **Store + scan:** built/pulled image is stored in a registry and **vulnerability-scanned** (Trivy/Grype-class) before it is allowed to run.
- **Run:** image is flattened to an ext4 rootfs and booted as a Firecracker microVM (**Option A**), with Nook supplying the kernel + a minimal init that reads the OCI config (Entrypoint/Cmd/Env/WorkingDir) and execs the app. **No container runtime / CRI in the serving path.**
- To a Colleague, a Tier 2 app is indistinguishable from a Tier 1 app.

### 7.3 Deploy input methods — [LOCKED]

- **Primary — agent-native CLI / MCP:** the builder's agent calls a Nook deploy tool and pastes back a URL. No dashboard, no git required. This is the "agent deploys for you" magic moment.
- **Secondary — drag-and-drop upload:** upload a folder in the browser, get a URL. The non-technical fallback.
- **[OPEN]** Git-push-to-deploy and a browser editor are explicitly deferred (candidates for later phases).

### 7.4 Data store — [LOCKED]

- **Zero-config SQLite-per-app** (LibSQL/Turso-style) in v1.
- Auto-provisioned on first deploy; connection injected as an **environment binding** — no user setup, no credentials to manage.
- Isolated per app; backed up by the platform.
- **[PROPOSED]** KV and managed Postgres as later add-ons for apps that outgrow SQLite.

### 7.5 URLs, networking & sharing — [LOCKED]

- Each app gets an instant URL: **[PROPOSED]** `https://<app>.<team>.nook.<tld>`.
- One URL fronts the whole app: `/` → static UI, `/api/*` → backend microVM.
- **Scale-to-zero:** idle apps are paused/snapshotted; next request restores/relaunches. Small software is mostly idle, so idle cost ≈ zero.
- **Sharing model:** a link + permission scope (see §7.6).

### 7.6 Auth & permissions — [LOCKED]

- **Identity:** **Google/Microsoft SSO** (Workspace / Entra). Enables the true Google-Doc move — **"anyone in my org"** scoping via verified email domain.
- **[PROPOSED] Permission scopes per app:**
  - *Private* — only the builder.
  - *Specific people* — named users/emails.
  - *Anyone in my org* — any verified member of the builder's domain.
  - *Public link* — anyone with the URL (off by default).
- **[PROPOSED] Roles:** Viewer/User (can use the app) vs. Editor (can redeploy) vs. Owner.
- **[OPEN]** SAML, SCIM, and audit logs → Enterprise tier, later.

### 7.7 Secrets & environment — [PROPOSED]

- Simple per-app secrets UI + CLI (`nook secrets set KEY=...`), injected as env vars at runtime (most tools call an external API or an LLM).

---

## 8. Architecture

### 8.1 High-level

```
                         ┌─────────────────────────────────────────────┐
   Builder + Agent  ───►  │  Product surface (TypeScript)                │
   (CLI / MCP)            │  • Dashboard (React)  • SDK/CLI  • REST API  │
   Drag-drop  ─────────►  └───────────────┬─────────────────────────────┘
                                          │
                                          ▼
                         ┌─────────────────────────────────────────────┐
                         │  Control plane (Go)                          │
                         │  • Deploy orchestrator  • Scheduler          │
                         │  • Build coordinator    • Router/gateway     │
                         │  • Auth/permissions     • Data provisioning  │
                         └───────┬───────────────────────┬──────────────┘
                                 │                        │
                 Tier 1 build    │                        │  Tier 2 build
             (detect → rootfs)   ▼                        ▼  (Dockerfile/img → rootfs, sandboxed + scanned)
                         ┌─────────────────────────────────────────────┐
                         │  Unified runtime: Firecracker microVMs        │
                         │  on Fly Machines  (image/rootfs → microVM)    │
                         │  • per-app isolation  • snapshot/scale-to-zero │
                         └───────────────┬─────────────────────────────┘
                                         │
                         ┌───────────────┴───────────────┐
                         │  SQLite-per-app  •  Secrets     │
                         │  •  Object/edge static assets    │
                         └─────────────────────────────────┘
```

### 8.2 Components — [LOCKED stack: TypeScript + Go]

- **Product surface — TypeScript:** React dashboard, the **CLI + SDK** (shared types), and the public REST API. Chosen for speed and to match the JS-heavy agent-app ecosystem.
- **Control plane — Go:** deploy orchestrator, scheduler, build coordinator, router/gateway, auth, data provisioning. Chosen for a robust, concurrent infra core.
- **Runtime substrate:** **Fly Machines** (managed Firecracker microVMs) — skips the hypervisor ops layer; Fly Machines *is* "image → microVM."

### 8.3 Deploy flow (both tiers)

1. Input arrives (agent CLI/MCP, or drag-drop).
2. **Tier 1:** detect shape → build rootfs from source. **Tier 2:** build (sandboxed) or pull image → scan → flatten to rootfs.
3. (Optional) pre-boot + **snapshot** for fast cold-start.
4. On request: launch/restore a **Fly Machine microVM**, attach rootfs + network + SQLite binding + secrets.
5. Router maps the app subdomain → the microVM.
6. Idle → pause/snapshot. Next request → restore.

### 8.4 Data model — [PROPOSED]

Core entities: `User`, `Org` (by SSO domain), `Team`, `App`, `Deployment`, `Runtime/Machine`, `DataStore`, `Secret`, `Permission`, `Membership`.

---

## 9. Security model (defense in depth) — [LOCKED principles]

1. **Sandboxed build** — building untrusted source/Dockerfiles happens in rootless BuildKit or an ephemeral microVM.
2. **Image/dependency scan** — vulnerability gate before run (Tier 2 especially).
3. **microVM boundary** — every app runs in its own Firecracker microVM with its own kernel (the core wall that makes non-technical arbitrary-code sharing safe).
4. **Jailer + seccomp** — contain even a compromised VMM.
5. **[PROPOSED] Egress policy** — default-deny outbound per app, allowlist as needed (stops data exfiltration / internal attacks).
6. **Auth on every URL** — SSO-gated by default; public is opt-in.

---

## 10. Key user flows

### 10.1 Builder ships a tool (happy path)
1. Builder finishes an app with their agent.
2. Says "deploy this and give me a share link."
3. Agent calls Nook's deploy tool → Nook detects, builds, provisions SQLite, boots a microVM → returns a URL.
4. Builder sets scope to "anyone in my org."
5. Builder pastes the link in Slack.

### 10.2 Non-technical colleague uses it
1. Opens the link → Google SSO → in.
2. Uses the tool. State persists (SQLite). Done.

### 10.3 Tier 2 custom environment
1. Builder's app needs a system binary.
2. Agent adds a `Dockerfile`; deploy via CLI with `--container` (or auto-fallback when Tier 1 detection can't satisfy deps).
3. Nook builds (sandboxed) → scans → runs as a microVM. Same URL/sharing/data experience.

---

## 11. MVP scope — [LOCKED: Full two-tier v1]

**In scope for v1:**
- **Tier 1** source deploy, **both runtimes** (TS/JS **and** Python), auto-detect + `smallapp.json`.
- **Tier 2** BYO container: **both** Dockerfile-from-source and prebuilt image, sandboxed build + scan, image→microVM.
- **Deploy inputs:** agent CLI/MCP **and** drag-and-drop upload.
- **Data:** zero-config SQLite-per-app.
- **Auth:** Google/Microsoft SSO with org-scoped sharing + permission scopes.
- **Runtime:** Firecracker microVMs on Fly Machines, with snapshot/scale-to-zero.
- **Secrets/env**, instant URLs, dashboard, CLI/SDK.
- **Open-source:** CLI + SDK + runtime conventions published.

**Out of v1 (deferred):** git-push deploy, browser editor, KV/Postgres, SAML/audit, egress-policy UI, on-prem/Kata, isolate-based cost optimization, marketplace/templates.

---

## 12. Milestones — [PROPOSED]

| Phase | Goal | Key deliverables |
|---|---|---|
| **M0 — Spike** | Prove image→microVM on Fly | Deploy a hardcoded app to a Fly Machine microVM, get a URL |
| **M1 — Tier 1 loop** | Source → URL | Auto-detect (TS/JS), CLI deploy, router, SSO login |
| **M2 — Stateful + shareable** | Real tools | SQLite-per-app, secrets, org-scoped permissions |
| **M3 — Python + drag-drop** | Broaden input | Python runtime, drag-and-drop upload |
| **M4 — Tier 2** | Custom envs | Sandboxed build (Dockerfile + image), scan, image→microVM |
| **M5 — Agent-native + OSS** | The magic + wedge | MCP deploy tool, published open-source CLI/SDK |
| **M6 — Polish + beta** | Design-partner ready | Dashboard, scale-to-zero tuning, onboarding |

---

## 13. Success metrics — [PROPOSED]

- **Activation:** % of new builders who ship a first app that gets ≥1 non-builder user.
- **Time-to-first-URL:** median seconds from deploy command to live URL.
- **Sharing coefficient:** avg. distinct users per app (are Colleagues actually using tools?).
- **Retention:** teams with ≥1 active app after 4 weeks.
- **Cost:** infra $/idle-app-month (validates scale-to-zero economics).

---

## 14. Future / out of scope

- **Enterprise on-prem** via **Kata Containers** as a K8s `RuntimeClass` (kept in back pocket).
- **firecracker-containerd** to borrow containerd's image caching/lazy-pull if DIY image handling hurts.
- **V8 isolates** as a cheaper runtime tier for the simplest static/JS apps.
- More runtimes, git-push, browser editor, templates/marketplace, richer datastores.

---

## 15. Open questions — [OPEN]

1. **Pricing specifics** (§5, §13) — free-tier limits, per-seat price, enterprise gating.
2. **URL scheme** (§7.5) — subdomain structure and custom domains.
3. **Permission granularity** (§7.6) — exact roles/scopes for v1.
4. **Data model details** (§8.4) — confirm entities/relationships before build.
5. **Secrets UX** (§7.7) — confirm inclusion in v1.
6. **Egress policy** — v1 default-deny or deferred?
7. **Design partners** — which 3–5 teams do we build M6 around?

---

## 16. Decision log

| # | Decision | Value |
|---|---|---|
| 1 | Product / name | **Nook** — cloud for small software |
| 2 | Audience | Small teams; builder wedge → all colleagues |
| 3 | Business model | Open-core SaaS (hosted business + OSS CLI/SDK/runtime) |
| 4 | Deploy model | Two tiers, unified on microVMs |
| 5 | Tier 1 | Source deploy, auto-detect, TS/JS + Python, `smallapp.json` |
| 6 | Tier 2 | BYO container — Dockerfile **and** prebuilt image → image→microVM |
| 7 | Isolation | Firecracker microVMs (Option A, no CRI in serving path) |
| 8 | Host | Fly Machines |
| 9 | Deploy input | Agent CLI/MCP + drag-and-drop |
| 10 | Data store | Zero-config SQLite-per-app (v1) |
| 11 | Auth | Google/Microsoft SSO, org-scoped sharing |
| 12 | Platform stack | TypeScript (product surface) + Go (control plane) |
| 13 | MVP scope | Full two-tier v1 |
