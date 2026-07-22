# Entity Lifecycle — Keeping Entities in Sync

Entities are an **ongoing mirror of the vendor's own source of truth**, not a one-time import. Whatever hierarchy you govern lives in *your* system first (an admin DB, an IdP, a project service); the governance tree is a projection of it. If you upsert entities once at onboarding and never touch them again, the tree drifts: renamed units keep stale display data, deleted units keep consuming budget, and restructures never reach enforcement. **Wire entity/assignment ops into the same create / update / delete / re-parent code paths that mutate your source of truth** — treat them like any other write that must stay consistent.

The entity model is **arbitrary**: govern any hierarchy your product needs (cost centers, projects, regions, tenants, sub-accounts, agents). The user / team / workspace / org examples below are *illustrative only* — never the required model. Pick the levels that actually carry budgets (max 4 deep, see `entity-model.md`).

## Vendor event → governance operation

Each source-of-truth event maps to a governance operation — all idempotent, safe to replay:

- **Registers/creates** a unit (new team, project, sub-account) — Entity **upsert** (bulk `PUT`) — add the entity; set `parentId` on its **`upsertAssignment`** to place it in the tree.
- **Renames / changes attributes** of a unit — **Re-upsert** the entity (patch `metadata`) — same id, new fields.
- **Deprovisions / deletes** a unit — **Archive** the entity (`.../entities/archive`).
- **Restructures / moves** a unit under a new parent — Set the new `parentId` on the entity's **`upsertAssignment`** (see re-parent rules below).

Because upsert is a bulk, idempotent `PUT` keyed by id, the cleanest sync is often **declarative**: on each sync tick (or each source-of-truth change event / webhook), push the *desired* set of entities and assignments and let Stigg reconcile. You don't have to diff — re-submitting the same payload converges to the same state.

## The three lifecycle gotchas

These are where a naive mirror breaks. All three are verified against the live surface.

### 1. Placing / moving a node — set `parentId` on `upsertAssignment`

To place or move a node in the tree, set `parentId` on the entity's **`upsertAssignment`** call. `parentId` is the entity's **single parent** — one position per entity, not a per-capability value. It is **tri-state**:

- **omit** → leave the current parent unchanged (a brand-new node defaults to a root);
- **`null`** → detach to a root;
- **an entity id** → set or change the parent.

The assignment payload is flat (one row per entity × capability × scope), so if the same entity appears in several rows, `parentId` is the **same** value on each — it's the node's parent, not that row's. Practical note: the *entity* upsert rejects `parentId` (`Unrecognized key(s) in request: "parentId"`), so parenting happens on `upsertAssignment`. Other assignment fields you omit (`usageLimit`, `cadence`) are preserved. Full field reference: `assignments.md`.

### 2. Re-parenting is leaf-only — and a non-leaf move fails ugly

**You can only re-parent a leaf node.** Moving a node that still has children is rejected — and on the current surface it surfaces as an opaque **HTTP 500**, not a clean 4xx. So when you restructure a subtree, **re-parent bottom-up**: move (or archive) the children first, then the former parent becomes a leaf and can be moved. Don't retry the 500 as if it were transient — it's the leaf-only rule.

### 3. Archive is *not* leaf-gated — it will orphan children

Archiving an entity is **allowed even when it has children**: the parent gets archived and its children are left **active but orphaned** (their assignments still point at a now-archived parent). There is **no** "archive the leaves first" guard here — that constraint belongs to re-parenting (gotcha 2), not archive. So a correct deprovision-a-subtree sync must **archive bottom-up itself**: archive/re-home the descendants first, then the parent — the platform will not stop you from stranding a subtree. Archived entities drop out of the default entity list; `unarchive` restores them.

> **What happens to an archived entity's unused budget?** There is **no** automatic re-distribution of an archived entity's unused budget to its parent — a limit sits **directly on the entity**, so archiving simply stops that entity accruing. Don't rely on any allocation-return behavior.

## Attribution keys are an append-only contract

Usage attributes to an entity via the entity type's `attributionKeys` matched against event/usage `dimensions`. There is **no retroactive attribution**: events reported before a dimension existed never back-fill. So when your source of truth introduces a new attribution dimension, treat the keys as append-only and start emitting the dimension on every relevant ingest call going forward (see `enforcement-and-usage.md` for the ingest fork and attribution mechanics).
