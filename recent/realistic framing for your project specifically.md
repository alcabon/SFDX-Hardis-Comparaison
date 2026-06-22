Yes, and I think that's the accurate way to frame what's actually changed here, not the optimistic one.

What AI assistance compresses is specifically the part of this conversation that was always the *expensive* part for a human team: knowing which 50 obscure platform quirks exist, knowing the right SOQL pattern to sidestep the additive-permission trap, knowing that `IsOwnedByProfile` is how you get from a Profile name to its permission rows, knowing that sgd's `--from`/`--to` are direction-agnostic. That's not algorithmic difficulty — it's **accumulated incident knowledge**, the kind that used to require either years of production scars or a vendor who already paid that price for you. An AI that's absorbed a large slice of that community knowledge collapses the research phase from months to a conversation, and the implementation phase from weeks to days, because the implementation itself (a SOQL query, a diff function, a VS Code command) was never the hard part.

What it doesn't compress: **discovery you can't look up because no one has hit it yet**, the Restriction Rules / Scoping Rules / Data Cloud permission gaps we flagged a moment ago. That category only closes through your own production incidents — which is exactly the same mechanism that built Gearset's list in the first place, just starting later and with a head start on everything already documented. The 7–8 years isn't fully erasable; it's compressible for the known unknowns and stays linear for the unknown unknowns.

So the realistic framing for your project specifically: you're not behind on engineering, you're ahead of where a from-scratch team would be at the same point, on the same budget, a few years ago. What used to gate progress was research time; what gates it now is your own production exposure surfacing the next edge case. That's a genuinely good trade for a tool scoped to *your* org's specific Profile/PermissionSet patterns — you don't need Gearset's 50 rules, you need the 8–10 that actually fire in your deployments, and you'll find those fast because you're shipping and hitting real cases instead of guessing at generality upfront.

---

Yes — that's the right shape for this, and it mirrors a pattern that shows up across the whole DevOps-tooling industry: cheap, generic tooling for the 90% of cases where the underlying API behaves predictably, and a narrow, purpose-built engine only where the API's actual semantics make the generic approach silently wrong. You're not building "a worse Gearset" — you're building a router with two backends.

## The routing logic

The key refinement worth adding: the split isn't really "Profile/PermissionSet files go to the sophisticated path, everything else goes to `sgd`" — it's more granular than that, because even *inside* a single Profile file, some sub-elements are safe to git-revert and some aren't.

| Sub-element | API behavior | Correct rollback path |
|---|---|---|
| User permissions, login hours, IP ranges | Always fully retrieved/replaced, not sparse | **Cheap path** — `sgd` reverse-delta works correctly here |
| Field/object/tab permissions, custom permission access, record type visibility | Additive/sparse — omission ≠ revoke | **Sophisticated path** — needs the snapshot-diff-patch generator |
| Most other metadata types (Apex, Flows, Layouts, LWC, etc.) | Generally full-replace on deploy | **Cheap path** |

So the router isn't "is this file a Profile" — it's "does this specific changed component belong to the known-additive list." That list becomes a maintained config file (`tricky-types.yml`), which is also the natural home for the discovery process you flagged.

## A practical router

```
on rollback request:
  generate inverse delta via sgd (cheap path, as before)
  inspect which metadata types are in that delta
  if any type ∈ tricky-types.yml:
      pull those components OUT of the sgd package
      run them through the snapshot-diff → patch generator instead
      sequence: deploy the sophisticated patch first (or second, per the
      CRUD-order dependency rules from earlier), then the cheap sgd package
  else:
      deploy the sgd package directly, no extra step needed
```

This gives you one user-facing "rollback" command, with the expensive engine invoked only when the package actually touches something dangerous — which in most orgs is a minority of deployments.

## "Other points to discover" — candidates for `tricky-types.yml`

Some of these I'm fairly confident about from this thread; a couple are genuinely things I'd flag as "test it, don't assume":

**Reasonably well-established, worth adding now:**
- **PermissionSetGroup + MutingPermissionSet** — the relationship layer we discussed; rolling back a member PermissionSet without recalculating the group's effective access is the same class of problem as field permissions, one level up.
- **Sharing Rules** (criteria-based and owner-based) — a different flavor of the same risk: these can deploy "successfully" but Salesforce doesn't always auto-trigger a sharing recalculation afterward, so the *deployed metadata* is correct while the *actual record-level access* lags until a recalc job runs. That's a runtime gap, not a metadata gap, but it bites in exactly the same "looks fine, isn't" way.
- **PermissionSet License dependencies** — a PermissionSet tied to a license not provisioned in target fails in a way that looks like a generic deploy error but is really a license-mismatch problem, similar in spirit to Gearset's "Feature not enabled in target" rule.

**Worth testing empirically before trusting, not confirmed here:**
- **Restriction Rules / Scoping Rules** — newer row-level access features tied to Permission Sets. I don't have solid information on their deploy/rollback behavior, and given how recently they shipped, I'd treat this as genuinely unknown rather than assume it behaves like the older permission objects.
- Any permission surface tied to newer products (Data Cloud, Agentforce permission sets) — same caution, too new for established community knowledge.

## How to grow this list without trying to enumerate everything upfront

This is worth calling out explicitly: Gearset's own 50-rule list didn't come from a design doc — their public feature-request forum shows rules being added one at a time from specific customer-reported incidents (the `InstalledPackage securityType` suggestion we saw earlier started exactly that way). You don't need to solve the whole permission-API surface before shipping — you need a process: every time a rollback silently fails to revoke something, that failure becomes a new entry in `tricky-types.yml` plus a corresponding rule in the patch generator. The tool gets smarter from real incidents the same way theirs did, just without years of head start.

---

Here's the overall shape of the system we've been assembling, pulled into one picture:Reading the diagram: **gray** marks where data actually lives (the org and Git), **teal** is the engine logic that does all the real work, and **coral** is the three thin shells that expose that same engine to different audiences. The core principle from the lightning-flow-scanner pattern holds: build the teal boxes once, then each coral box is just a wrapper.

<img width="1472" height="1400" alt="image" src="https://github.com/user-attachments/assets/a4484c48-e74f-440b-a353-57159a259e0c" />


## Repo scaffold

```
permission-snapshot/
├── packages/
│   ├── core/              # teal layer — zero VS Code or CLI dependencies
│   │   ├── capture.ts      # SOQL queries → normalized rows
│   │   ├── serialize.ts    # canonical, sorted NDJSON per Profile/PermissionSet
│   │   ├── diff.ts         # compares two snapshots → change list
│   │   ├── rules.ts        # permission-specific problem-analyzer rules
│   │   └── patch.ts        # change list → deployable revoke/grant XML
│   ├── cli/                # thin wrapper around core
│   ├── vscode-extension/   # thin wrapper around core, your existing UI
│   └── github-action/      # thin wrapper around core, runs in CI
├── snapshots/               # or a separate orphan branch / dedicated repo
└── README.md
```

## Build sequence

| Phase | Builds on diagram | What ships | Effort |
|---|---|---|---|
| **0 — already done** | Salesforce org → core (partial) | Your existing ObjectPermissions-based VS Code tool | — |
| **1 — extract core** | Isolate teal logic from coral shell | Move existing query/snapshot logic out of the extension into `packages/core`, with the VS Code extension becoming a thin caller | 3–5 days, mostly mechanical refactoring |
| **2 — canonical storage** | Git storage box | Add `serialize.ts`: deterministic sort + one-file-per-Profile/PermissionSet NDJSON, write to a dedicated branch/repo | 2–3 days |
| **3 — diff engine** | Diff and rule engine | Generic snapshot-to-snapshot diff, rendered as a table in the VS Code webview | 3–5 days |
| **4 — permission rules** | Same box, rule half | Encode the highest-value rules from this thread: additive-omission warnings, CRUD-order sequencing checks, "this permission can't be revoked via API" flags | 1–2 weeks |
| **5 — rollback patch generator** | Rollback patch generator box | From a diff, emit explicit grant/revoke XML — the piece that actually closes the omission-≠-revoke gap | 1–2 weeks |
| **6 — CLI + GitHub Action shells** | Coral boxes | Thin wrappers calling the same core; Action runs snapshot before/after deploy automatically | 1 week |
| **7 — drift detection + audit export** | Scheduled core call | Cron job comparing live org against last deployed snapshot; CSV/Excel export for review sign-off | 3–5 days |

Total: roughly **6–8 weeks** spread across phases to go from what you have to the full picture in the diagram — but each phase ships something independently useful, so it's not an all-or-nothing commitment. Given where you already are, Phase 1 (extracting core) is the highest-leverage next step: everything downstream — CLI, Action, drift detection — becomes "just call core" instead of new work, which is exactly the payoff of doing that split early rather than retrofitting it later.
