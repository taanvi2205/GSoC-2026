<div align="center">

# Safe Archival & Restore for SW360

### Reversible archival & restore of stale entities @ Eclipse SW360

**Google Summer of Code 2026**

<!-- badges (optional): add view counter / GitHub follow badges here -->
<img width="170" height="170" alt="images (1)" src="https://github.com/user-attachments/assets/01a0fca3-0fd4-4fe9-8847-f3a8ae9280ca" />


</div>

---

## Project Details

**Organization** - [Eclipse SW360](https://www.eclipse.org/sw360/)

**Contributor** - Taanvi Khevaria ([@taanvi2205](https://github.com/taanvi2205))

**Mentors** - Gaurav Mishra ([@GMishx](https://github.com/GMishx)), Amrit Kumar Verma ([@amritkv](https://github.com/amritkv)), Farooq Fateh Aftab ([@Farooq-Fateh-Aftab](https://github.com/Farooq-Fateh-Aftab)), Akshit Joshi ([@akshitjoshii](https://github.com/akshitjoshii))

**Proposal** - [GSoC 2026 Proposal – Archival & Restore](https://drive.google.com/file/d/18xpSIQbhrpp5MGQQS6MzLGlncXtwPMcB/view?usp=sharing)

**Repos** - [eclipse-sw360/sw360](https://github.com/eclipse-sw360/sw360) (backend), [eclipse-sw360/sw360-frontend](https://github.com/eclipse-sw360/sw360-frontend) (frontend)

> **Contributions:** 9 pull requests across the backend and frontend <!-- fill exact count/lines from GitHub -->

---

##  What's this project actually about?

Over time SW360 instances fills up with entities nobody uses anymore like abandoned **projects**, old **releases**, orphaned **packages**. The only way to get rid of them was **delete**, which is destructive and forgetful because the data is gone, with no record that it ever existed or why it left.

This project adds a safer middle path: **archival** which take a stale entity *out* of the live database, keep a complete, self-contained copy as a downloadable `.tar.gz` archive, keep an audit record of what happened, and be able to **restore** it later, exactly as it was.

The guiding principle: **archiving should feel as safe as moving a file to a folder you can always reopen, not deleting it.**

---

## Architecture

```
                          ARCHIVE                                    RESTORE
   ┌───────────────┐                                   ┌───────────────┐
   │  Admin (UI)   │  select entities → Archive        │  Admin (UI)   │  upload bundle → Restore
   └───────┬───────┘                                   └───────┬───────┘
           │  /resource/api/archival/*                         │  /resource/api/archival/restore
   ┌───────▼───────────────┐                           ┌───────▼───────────────┐
   │   Resource-server     │  (gateway, auth/token)    │   Resource-server     │
   └───────┬───────────────┘                           └───────┬───────────────┘
   ┌───────▼───────────────┐                           ┌───────▼───────────────┐
   │  Archival service     │                           │  Archival service     │
   │  ArchiveBuilder       │  stream TAR.GZ  ─────────▶│  BundleReader         │
   │  Sw360EntityProvider  │                    ▲      │  RestoreHandler       │
   └───────┬───────────────┘                    │      └───────┬───────────────┘
           │ collect + delete                   │              │ presence-driven re-insert
   ┌───────▼─────────────-──┐                   │      ┌───────▼───────────────┐
   │      CouchDB           │◀──────────────────       │      CouchDB          │
   │  projects/components/  │   .tar.gz downloaded     │  (original ids,       │
   │  releases/packages     │   to the admin's disk    │   stripped revisions) │
   └────────────────────────┘                          └───────────────────────┘
                    │                                             ▲
                    └──────── Archival registry (audit ledger) ───┘
```

**Bundle format.** Every archive is a single `.tar.gz` file. At its root there is a
`manifest.json` describing what the bundle contains. Each archived entity
contributes its document as `{type}.json`, and each of its attachments as a
pair of files, `{attId}.bin` for the bytes and `{attId}.meta.json` for the
metadata. The manifest records a SHA-256 checksum per entity, so restore can
verify the contents before writing anything.

**Registry.** Separately, each archived entity gets an `ArchivalRecord` - who
archived it, when, and why, plus a status that moves from `ARCHIVED` to
`RESTORED`. These records live in their own database, deliberately apart from
the bundles: the registry is the audit ledger (*that* something was archived)
and stays small and queryable, while the bundle carries the data (*what* it
was) and stays portable.

---

## Contributions
### Phase 1 - Backend archival service 
A new Spring Boot module, `backend/archival` (Java 21). It packages a stale entity into a downloadable file, then removes it from the live database.

- **`ArchiveBuilder`** writes the `.tar.gz` directly to the output stream. Nothing is held in memory, so archive size doesn't matter. It computes a SHA-256 per entity as it writes.
- **`EntityProvider`** reads the data. `Sw360EntityProvider` talks to the live databases; `InMemoryEntityProvider` stands in for CouchDB during unit tests.
- **`ArchivalHandler`** runs the sequence for all four entity types: write an audit record, collect the entity, stream the bundle, delete the original through SW360's existing delete pipelines.
- **Nothing gets orphaned.** A release which another live project still uses is bundled but not deleted. A package whose parent release is still live is refused outright.
- **A preview before actual archival.** A dry run shows what would be archived, kept, or blocked first. The whole feature is admin-only, checked twice.

→ **PR [#4336](https://github.com/eclipse-sw360/sw360/pull/4336) - merged**

### Phase 2 - Restore 
The other half of the project: put an archived entity back exactly as it was.

- **`BundleReader`** is `ArchiveBuilder` in reverse. It unpacks an uploaded `.tar.gz` into the manifest and one object per entity, keeping the original file order as the checksums depend on it.
- **`RestoreHandler`** checks the live database first. Anything already there is skipped; everything else is re-inserted under its **original id**, with the CouchDB revision stripped. Because it skips what exists, restore is safe to run twice.
- **Order matters.** Releases go back before the components, projects, and packages that point at them.
- **Attachments come back byte-for-byte**, under their original content-ids.
- **Checksums are verified before anything is written.** Each entity's SHA-256 is recomputed from the bundle and compared to the manifest. A corrupt or edited bundle fails that entity instead of quietly writing bad data.
- Covered by unit tests (re-insert, skip-existing, ordering, round-trip, deliberate tamper) and verified end-to-end against a live CouchDB.

→ **PR [#4453](https://github.com/eclipse-sw360/sw360/pull/4453)**

### Phase 3 - Resource-server routing 
The archival service runs on its own. This phase puts it behind SW360's public REST API so the frontend never calls it directly.

- A typed client in the datahandler library (`ArchivalClient` / `ArchivalServiceRestClient`) forwards archive, preview, restore, and records calls.
- `Sw360ArchivalService` and a `@BasePathAwareController` expose them under `/resource/api/archival/*`.
- Archive streams the `.tar.gz` back to the caller; restore takes a multipart upload.
- The acting user comes from the authenticated principal, never from the request body.

→ **PR [#4435](https://github.com/eclipse-sw360/sw360/pull/4435) + [#4460](https://github.com/eclipse-sw360/sw360/pull/4460)** 

### Phase 4 - Frontend archive UI 
Archiving used to be a per-row icon and a button on every detail page. It's now one bulk action on the list pages.

- Row checkboxes (TanStack Table) on the projects, components, and packages pages.
- A two-step button: **Archive** reveals the checkboxes and becomes **Confirm**. **Cancel** backs out.
- The modal shows the plan before you commit like how many will be archived, kept, or blocked, and a table of each one. A comment is required.
- The resulting `.tar.gz` downloads straight to the browser.
- Old entry points removed from detail pages and row actions; everything routed through the resource-server.

→ **PR [#1898](https://github.com/eclipse-sw360/sw360-frontend/pull/1898)**

<img width="1468" height="786" alt="Screenshot 2026-08-21 at 1 36 39 AM" src="https://github.com/user-attachments/assets/2accc6d7-2075-47a6-bc44-2acfe22b3710" />
<img width="1466" height="800" alt="Screenshot 2026-08-21 at 1 36 59 AM" src="https://github.com/user-attachments/assets/d27ef8c0-019a-4f75-8882-754c3d8ce660" />

### Phase 5 - Frontend restore UI 
A dedicated admin page at `/admin/archival`, linked from the Admin dropdown.

- **Upload** the `.tar.gz` you downloaded when you archived.
- **Preview** shows which entities would come back and which are already present.
- **Restore**, then a per-entity result table. Anything that failed appears in a box with the reason.
- Below that, a registry of archival records - type, status, who and when, each deletable.
- Talks to the resource-server through a typed service layer, with multipart upload for the bundle. Translated into **10 locales**.

→ **PR [#1971](https://github.com/eclipse-sw360/sw360-frontend/pull/1971)**

<img width="1467" height="796" alt="Screenshot 2026-08-21 at 1 44 02 AM" src="https://github.com/user-attachments/assets/eedbd38c-789d-4f91-9079-c894de92d7ff" />
<img width="1470" height="800" alt="Screenshot 2026-08-21 at 1 44 23 AM" src="https://github.com/user-attachments/assets/1812ac02-51e9-40d8-9998-eb6e7c8e32a0" />

### Phase 6 - Final touches
The closing phase tightens keep-alive handling, makes previews accurate, and prevents silent duplicate creation.

* **Complete keep-alive:** Releases survive component archival if another live release still references them. Package archival is blocked if another live project uses the package.
* **Duplicate-aware creation:** Creation forms now show previously archived entities of the same type, with a *contact admin for restoration* note. Added `/records/search` to support this.
* **Cleanup:** Removed the unused `includeChangelogs` flag.

→ **PR [#4491](https://github.com/eclipse-sw360/sw360/pull/4491) + [#4515](https://github.com/eclipse-sw360/sw360/pull/4515) + [#1982](https://github.com/eclipse-sw360/sw360-frontend/pull/1982)**

---

## Key Design Decisions

- **Reversible by construction.** Everything needed to reconstruct an entity like its document, its attachments, and a manifest goes into one portable TAR.GZ. Restore re-inserts with the *original id* and a stripped revision, so an archive can always be undone.
- **Presence-driven restore.** If an entity already exists it is skipped; otherwise re-inserted. This makes restore **idempotent** and safe to re-run.
- **Dependency-aware keep-alive.** A release/package still referenced by another live entity is kept alive, never orphaned, so archiving one thing never silently breaks another.
- **Streaming, not buffering.** Bundles (entity JSON + attachment blobs) stream straight to the client, so archive runs in constant memory regardless of size.
- **Integrity at the boundary.** The bundle is the only data that leaves and re-enters SW360, so it carries per-entity SHA-256 checksums that are verified on restore, before any write.
- **Registry ≠ bundle.** A small metadata ledger records *that* something was archived (audit trail); the bundle holds *what* it was (the data). The ledger stays queryable; the bundle stays portable.
- **Admin-only, defense in depth.** Enforced at the controller and re-checked deeper in.

---

## What's left / Open Questions

**Future work:**
1. **Full dependency-closure keep-alive** - extend from the release↔project edge to sub-projects and shared-attachment reference-counting; move from a point-in-time check to a proper reference-counted model.
2. **Open PRs** (in review): restore backend (#4453), resource-server routing (#4435 + restore routing), shared-dependency keep-alive, frontend archive UI (#1898), frontend restore UI. These land as the base Thrift-removal branch and reviews progress.

---

## What GSoC Taught Me

Building reversible archival and restore for Eclipse SW360 taught me a lot beyond the feature itself.

* **Design for failure early.** Keeping original IDs, validating checksums, and storing enough information made reliable restoration possible.
* **Architecture matters.** Streaming archives and idempotent restore made the system safer and more scalable.
* **Large codebases take patience.** Working with SW360 and the ongoing Thrift-to-Spring migration taught me to adapt to changing code.
* **Reviews improve the work.** Mentor feedback helped me catch edge cases and make better design decisions.
* **Break big problems down.** Splitting the feature into six phases made it easier to build, test, review, and merge.

---

## Acknowledgements

Huge thanks to my all mentors for patient and specific reviews, clearing up my smallest of doubts, and to the **Eclipse SW360** community for a welcoming first open-source project.
