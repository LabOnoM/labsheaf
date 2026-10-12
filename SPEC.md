# Labsheaf protocol

| | |
|---|---|
| Version | Draft 0.2, 2026-10-12. Not yet stable: formats may change before version 1.0. |
| Licence | MIT, copyright Nexitia LLC (see `LICENSE`). |
| Scope | Serverless collaboration on structured analysis documents (a node graph and its annotations) through a folder that every collaborator can read and write. |
| Status | Specification only. A conformance suite and a reference implementation will follow in this repository. |

## 1. Purpose and properties

Several machines edit one structured document through a folder they can all read and write, online or offline, with no
server in the path.

- **Convergence.** Every machine that has accepted the same batches computes the same document, byte for byte.
- **Authorship and integrity.** Every change is signed by an admitted machine of its author's account. Any reader checks
  this offline.
- **Independence.** Once a machine is admitted to a document, its changes stay valid whatever happens to billing,
  certificates or the issuer. A document keeps working if its issuer is unreachable for years or closes.
- **Openness.** Files are plain JSON, readable without any particular software.
- **Tolerance of real storage.** It works over SMB, NFS and synced folders whose files arrive late, out of order, twice,
  or beside "conflicted copies" made by sync tools.
- **Not covered.** Keystroke-level live typing; confidentiality, which belongs to the folder's access control; deciding
  what is scientifically right, which stays with people through visible conflicts.

## 2. Terms

| Term | Meaning |
|---|---|
| Document | One shared document: a folder with the layout of section 3. |
| Machine | An installation with one key pair, identified by a device id. |
| Admission | A signed statement in the document that a machine's key belongs to a member's account (section 10). |
| Member | An account with a role in the document: owner, editor, commenter or viewer. |
| Issuer | A service that certifies that a machine key belongs to an account; nexitia.ai runs one, and a lab may run its own. |
| Overlay | The annotation layer beside the graph: groups, phases, sticky notes and review threads. It never changes computed results. |
| Batch | One write-once file holding a machine's signed changes. |
| Snapshot | A compacted, signed state that lets a reader skip old batches. |
| Replica | The full local copy each machine keeps, with an outbox of signed batches not yet in the folder. |

## 3. Folder layout

```text
<title>.<product-extension>/            the document; on macOS registered as a package so it shows as one file
  manifest.json                         written once and signed by the creator's machine (section 10)
  devices/<device-id>/
    head.json                           the latest batch number and hash; replaced by its own machine only
    certs/<sha256>.json                 this machine's certificates, as issued (public)
    log/<seq:012d>-<hash12>.batch.json  write-once signed batches, numbered from 1 without gaps
    log/archive-<from>-<to>-<hash12>.json   optional: the same signed batches gathered verbatim (section 11)
  snapshots/<hash12>.json               compacted states, content-named
  notices/<hash12>.json                 issuers' signed compromise notices, relayed by any member (section 10)
  names/<device-id>.json                display name of the machine's account, signed, replaceable (section 18)
  presence/<device-id>.json             optional heartbeat, replaced by its own machine only (section 13)
  cache/<aa>/<lineage-hash>/<device-id>/   optional results, per writer, with a signed list of file hashes (section 14)
```

Names use lowercase ASCII letters, digits, `-` and `.` only, at most 64 characters each, so they behave the same on
case-insensitive SMB and macOS volumes, case-sensitive NFS exports and Windows path limits. Device and document ids are
random 128-bit values in lowercase hexadecimal. `<hash12>` is the first 12 hexadecimal digits of the SHA-256 of the
file's canonical bytes; a reader ignores a file whose bytes do not match its name.

## 4. Writing rules

1. **Names carry content hashes.** Two files with the same name therefore hold the same bytes, so a rename that
   replaces an existing file is harmless, and nobody can occupy a name another machine will need, because nobody can
   predict its hash. This makes write-once hold on every file system, whatever its rename semantics.
2. **One writer per device folder.** A machine writes only under `devices/<its id>/`, `names/<its id>.json` and
   `presence/<its id>.json`. Content-named files elsewhere may be written by any member.
3. **Sign once.** A batch is signed once and its signed bytes are stored in the outbox before the first attempt to write
   it; every retry writes those same bytes. A sequence number is never signed twice.
4. **One counter per machine.** The GUI, CLI and agent server of one machine share the device id, so they take sequence
   numbers under the machine's OS-level lock on the local disk (never on the share).
5. **Temporary name, then rename.** Every file is written as `.tmp-<random>` in its destination folder, flushed, then
   renamed. Appending is never used; it is not safe on NFS.
6. **`head.json` last.** After a batch lands, its machine replaces `head.json` with the new number and hash.
7. **Cadence.** A batch at most every two seconds while someone edits, and when editing pauses.
8. **Permissions.** On POSIX file systems, files are readable and writable by the group and never by everyone, and
   folders inherit the group; on SMB, files inherit the folder's access list.
9. **Caps.** Every file type has a size cap; a writer never exceeds it and a reader refuses anything larger.

| File | Cap |
|---|---|
| Batch | 1 MB |
| Archive, snapshot | 64 MB |
| Certificate, name, presence, head | 8 KB |
| Compromise notice | 256 KB |
| Results-cache file list | 1 MB; payload files as the product declares |

## 5. Reading rules

1. **Heads first.** Each poll reads every admitted machine's `head.json` and fetches only the exact batch names that are
   new. Full folder listings happen on open and every few minutes, to find new machines and notices. SMB and NFS do not
   announce other machines' changes, and NFS can hide a new file from listings for up to about a minute by default.
2. **Admitted machines only.** A device folder is read only after its machine's admission (section 10) has been
   accepted; until then only its certificates are read, within their cap.
3. **Bounded work.** Each poll has a limit on files and bytes; the rest waits for the next poll.
4. **Ignore foreign names.** Anything outside section 3's grammar is ignored: temporary files, `._*` and `.DS_Store`,
   editors' backups, and sync tools' conflicted copies. A conflicted copy that is itself a validly signed batch from an
   admitted machine is evidence of a fork (section 10) and is read as such.
5. **Tolerate disorder.** A batch whose predecessor has not arrived waits. A duplicate is ignored. A file that fails to
   parse is retried with growing back-off, since it may still be arriving, and reported if it stays broken.
6. **Verify before applying.** Signature, admission, chain, clock and role checks (sections 6, 9 and 10) come before a
   batch touches the document. A batch that fails stays aside, is reported, and is never applied.
7. **Treat everything as untrusted input.** Strict schemas, the caps above, and limits on nesting and string length
   apply to every file read from the share (section 15).

## 6. Records

### 6.1 Canonical form and signatures

Records are JSON, serialized for hashing and signing with the JSON Canonicalization Scheme (RFC 8785), UTF-8. Signatures
are ECDSA P-256 with SHA-256 (ES256) over the canonical bytes of the record without its `sig` member.

### 6.2 Batch

```json
{
  "format": "1.0",
  "doc": "4f0c…", "device": "a91e…", "account": "8d4c…", "seq": 42,
  "prev": "sha256 of batch 41's canonical bytes",
  "wall": 1792800000456,
  "registry": "hash of the node-spec registry the writer validated with",
  "rules": 1,
  "seen": { "a91e…": { "seq": 41, "hash": "…" }, "c07d…": { "seq": 17, "hash": "…" } },
  "ops": [
    { "n": 0, "t": [1792800000123, 0], "kind": "node.set",
      "target": "n_5b2e…", "path": "/config/min_genes", "value": 200,
      "via": { "kind": "agent", "channel": "mcp", "model": "…" } }
  ],
  "sig": { "alg": "ES256", "value": "base64url" }
}
```

- `account` must equal the account of the machine's accepted admission. Every operation in the batch is authored by
  that account; there is no per-operation author field to forge.
- `via` is optional attribution that an agent acted through this account's machine. It never grants anything.
- `wall` is the writer's wall clock at signing. It never decreases along a machine's chain, and it may not be earlier
  than the `wall` of any batch the writer lists in `seen`, minus ten minutes.
- `seen` names, for each machine, the last batch the writer had applied, by number and hash. It never decreases along a
  machine's chain. It carries causality, and its hashes let readers tell when two machines saw different batches under
  one number.

### 6.3 Operation kinds

| Kind | Effect | Who |
|---|---|---|
| `node.add` | Adds a node with a new random id, its type and initial settings; a node's type never changes | Editors |
| `node.remove`, `node.restore` | Removes or restores a node; edges touching it become inactive, not deleted | Editors |
| `node.set` | Sets one value at a JSON pointer inside a node's settings, note, label or position | Editors |
| `edge.add`, `edge.remove` | Connects or disconnects two ports | Editors |
| `overlay.*` | Group, phase, sticky-note and thread changes; thread messages and events add-only | Editors; commenters for thread operations only |
| `member.set` | Sets an account's role, including `none` | Owners |
| `device.admit` | Admits a machine key for an account (section 10) | Owners, or an admitted machine of the same account |
| `device.accept` | Accepts set-aside batches of a machine up to a number (sections 9 and 10) | Owners |
| `device.distrust` | Rejects a machine's batches after a given number | Owners |
| `trust.add`, `trust.remove` | Changes the issuers whose certificates may support admissions; removal takes an effective point and never touches earlier admissions | Owners |
| `doc.set` | Document metadata such as title or presence policy | Owners |
| `doc.registry` | Adopts a newer node-spec registry with its declared migrations (section 16) | Owners and editors |

An unknown kind is kept, ignored by readers that do not know it, and makes those readers read-only for the document.

### 6.4 Clock

Each operation carries a hybrid logical clock stamp `t = [milliseconds, counter]`. A machine never issues a stamp lower
than any stamp it has accepted, so wrong clocks cannot reorder causally related changes. A machine does not let a
foreign stamp more than a day ahead of its own wall clock advance its clock; such a batch waits until local time
catches up, so one fast clock cannot drag every other machine forward. The total order of operations is
`(t[0], t[1], device id, batch seq, n)`. Wall time is for display and for the checks in 6.2.

## 7. Merge

The document is the fold of all accepted operations, in total order, over the newest usable snapshot.

- **Values.** For each node value, the operation last in total order wins. When the winner did not know of the loser
  (by `seen`, the two are concurrent), the losing value is kept in the conflict list for people to restore. A value
  overwritten by someone who had seen it is simply replaced.
- **Existence.** A node exists when the last of its `node.add`, `node.remove` and `node.restore` in total order says so.
  Changes to a removed node are kept as conflicts and apply again if it is restored.
- **Edges.** An edge is active when it was added and not removed later and both its nodes exist.
- **Overlay.** Thread messages and events are add-only, deletion markers win, and concurrent edits
  of one note keep both versions until someone picks one.

## 8. Repair, rules version 1

After merging, every machine applies the same deterministic repair with the same node-spec registry. Repair results are
derived on every machine and never written as operations, so machines never fight over them.

| Rule | When | Result |
|---|---|---|
| R1 Cycles | Active edges, taken in total order, would close a cycle | The edge that closes it is set aside |
| R2 Single inputs | More than one active edge reaches an input that accepts one | The earliest stays; later ones are set aside |
| R3 Port kinds | An edge joins ports of incompatible kinds | Set aside |
| R4 Missing ends | An edge's node is removed | Inactive until the node is restored |
| R5 Invalid settings | A value fails the node's schema | Kept; the node is marked invalid and cannot run |

Everything set aside or in conflict is listed with who did what and when. Nothing is silently lost; an editor resolves an
item with an ordinary operation.

## 9. Membership and roles

| Role | May write |
|---|---|
| Owner | Everything |
| Editor | Node, edge, overlay and registry operations; snapshots; results-cache entries |
| Commenter | Thread operations in the overlay only |
| Viewer | Nothing; reading is governed by the folder's permissions |

- The creator is the first owner, named in the signed manifest.
- An operation is valid when its batch's account held the needed role in the membership state that batch had seen.
- **Removal is deterministic.** A `member.set` to a lower role or `none` records, for each of the account's machines,
  the last batch the remover had seen. Later batches from those machines are set aside, not applied; owners may accept
  them with `device.accept` after reviewing them, for example when they were written offline before the person knew.
  The same rule covers a removed owner's own late operations, so changes made after a removal never apply unseen.
- Concurrent role changes for one account resolve to the lower role.
- Removals are applied in total order, and one that would leave the document without an owner is skipped. Two owners
  removing each other concurrently therefore end the same way on every machine.

## 10. Trust

### 10.1 Trust anchors

- An app trusts issuer roots built into it (for example an issuer's current and next roots) and any other issuer root the
  person has confirmed on that machine by fingerprint. A document's manifest names the issuers it accepts.
- A root signs its successor, so a document follows an issuer's root rotation without any action by its owners.
- The manifest is signed by the creator's machine and carries the creator's certificate. Every machine checks it
  against its own trust anchors, and on first open pins the manifest's hash and the document id in its replica; a
  manifest that later differs is treated as an attack and the document opens read-only with a warning.
- `trust.add` of an issuer the local app does not already trust needs confirmation by fingerprint on each machine.

### 10.2 Admission

- A machine joins a document by writing its certificate under `certs/` and being admitted with `device.admit`, signed
  by an owner or by an already admitted machine of the same account. Its batches wait, shown as "waiting for approval",
  until then.
- The admitting machine checks that the certificate chains to a trusted root, names the same account, carries the same
  public key, and is valid at its own wall time. Other machines check the admission's signature and the certificate's
  chain and key.
- Where no issuer is reachable or wanted (an air-gapped site, or after an issuer has closed), an owner may admit a
  machine by its key fingerprint alone, confirmed in person or by another channel. The admission records that it was
  vouched for by the owner.
- After admission the document trusts that key for that account until the account is removed, the machine is
  distrusted, or the key is shown to be compromised. Certificate expiry, a lapsed licence or an unreachable issuer never
  invalidates batches already admitted or written. Licences are enforced by the writer's own app, never by readers.
- A Google or other account takeover alone therefore cannot write into a document: the attacker's new machine still
  needs admission by an owner or by one of the account's existing machines.

### 10.3 Forks, compromise and distrust

- Two different validly signed batches from one machine with the same number are a fork. The machine is frozen at the
  last batch before the fork on every machine, both branches are set aside, and owners are told.
- Issuers publish signed compromise notices naming key thumbprints they believe stolen, nothing else. Any member's app
  copies the latest notice into `notices/`. A machine that learns of a compromised key holds that key's unapplied
  batches and alerts owners; an owner's `device.distrust` then settles which batches stand, the same way on every machine.
- Ordinary events, such as a released licence slot or a replaced laptop, produce no notice: the licence simply stops that app from
  writing, and owners remove machines they no longer want.

## 11. Snapshots and compaction

- An editor's or owner's app writes a snapshot after enough new operations: the canonical document, set-aside and
  conflict lists, the `seen` vector it covers, the rules version and the registry hash, signed by its machine.
- A reader starts from the newest snapshot it can trust and applies later batches. Whenever it holds the batches a
  snapshot covers, it verifies the snapshot by replay; a snapshot that differs is ignored and reported. Membership,
  admissions and trust are taken from a snapshot only when it was verified by replay or signed by an owner's machine;
  otherwise the document opens read-only until the batches or an owner's snapshot arrive.
- A machine may gather its own old batches verbatim into an archive and delete the originals 30 days later. Readers
  accept either form. Archives keep replay possible; deleting them is a lab's decision about its records.

## 12. Replica and offline work

Each machine keeps a full replica of every document it has opened, in its app-data folder, keyed by document id and
encrypted at rest with a per-machine key kept in the OS keychain. Edits go to the replica and the outbox first; the
outbox is written to the folder whenever the share is reachable. A replica not opened for a configurable period is
removed and fetched again from the folder when needed. If the folder is lost, any member's replica can recreate it; the
hash chains show what each replica holds.

## 13. Presence (optional)

When the document's owners allow it, a machine replaces `presence/<device-id>.json` every ten seconds while the document
is open: the account id, a stamp, the selected node and the node being edited, signed. Readers treat entries older than
30 seconds as gone. Presence shows when people work, so it is off unless owners turn it on.

## 14. Results cache (optional)

An editor or owner who computes a result may write it under `cache/<aa>/<lineage-hash>/<device-id>/` with a signed list
of the files and their hashes. A reader uses an entry only from an admitted editor or owner, only if its own computation
of the lineage hash, including backend and package versions, matches exactly, and only after checking the file hashes;
it shows who computed the result. The cache lives on the lab's storage and is never uploaded anywhere.

## 15. Data references and safety

- Settings that name files hold paths relative to the document folder with `/` separators. Absolute paths, drive
  letters, UNC paths and URLs are refused when written, and refused again before any file-system call when read, so a
  crafted path never makes a Windows machine contact a remote server.
- A path is resolved with symbolic links followed, then checked to lie inside the share or the user's configured data
  folders; outside them it is not opened without the user's confirmation. Output paths follow the same rule.
- Settings that carry expressions, queries, formulas or scripts are code-bearing. Received changes never start a
  computation, and a code-bearing setting changed by someone else is shown and confirmed before its first run. Query
  engines must not be able to read files or run commands through expressions in settings.
- Every file type read from a share is fuzzed in the reference implementation's tests.

## 16. Versions and extensions

- `format` is major.minor. A reader opens a newer minor version and ignores unknown fields; a newer major version, an
  unknown operation kind or a newer rules version makes the document read-only until the app is updated.
- A document records its current node-spec registry. `doc.registry` adopts a newer one with the app's declared
  migrations. A machine edits at the document's current registry, or at one its app declares compatible; otherwise it
  reads and names the update it needs.
- `encryption` in the manifest is reserved and must be `null` in this version.

## 17. Security considerations

| Threat | Protection | What remains |
|---|---|---|
| Someone writes changes in another person's name | Batches signed by admitted machine keys; the batch's account fixed by its admission | Nothing, unless a key is stolen |
| Someone swaps the manifest or its trusted issuers | Signed manifest checked against each app's own anchors; manifest hash pinned in replicas; local confirmation for new issuers | Nothing for existing members; a newcomer must get the document from a trusted source |
| An account is taken over at the identity provider | Admission by an owner or by an existing machine of the account | An owner who admits without checking |
| A removed member keeps writing | Removal freezes their machines at what the remover had seen; later batches set aside | Nothing applies without an owner accepting it |
| Files on the share are altered or deleted | Content-hash names, signatures and hash chains detect it; replicas restore | Deletion cannot be prevented, only detected and repaired |
| A machine key is stolen | Fork detection, compromise notices relayed through the folder, `device.distrust` | Batches the thief wrote before owners act, which owners can then reject |
| A non-member with write access to the share | Unadmitted folders are not read beyond capped certificates; file caps; bounded polling | Filling the disk or deleting files; the share's permissions defend against that |
| Anyone with read access reads the document | None by design: contents are plain | The share's permissions; replicas encrypted at rest on each machine |
| Crafted files | Strict schemas, caps, name grammar, fuzzing | Parser bugs not yet found |
| A crafted document reads private files or contacts a remote server | Path confinement after resolving links, refusal of UNC paths and URLs, no file reads from query expressions | A user who confirms |
| A crafted document runs code | No automatic runs; confirmation of code-bearing settings changed by others | A user who confirms |
| An issuer's root is compromised | Roots in hardware, successor signing, compromise notices, trust lists that can drop an issuer from a point on | Admissions an owner made with a forged certificate during the compromise |
| One fast clock drags others forward | Foreign stamps more than a day ahead wait | Nothing |
| Old batches are replayed | Batches are idempotent by machine, number and hash | Nothing |

## 18. Privacy considerations

- Signed records carry account ids, never names or emails, because a signed record cannot be edited later.
- Display names live in `names/`, which their own machine can replace or remove, so a person can take their name out.
- Certificates carry the account id, device id, public key, capabilities and validity: no name, email, organization
  list or purchase list. Machines need one at enrolment and when joining a document, so the folder holds a few
  certificates per machine, not a timeline of activity.
- Compromise notices carry key thumbprints only, with no device id, account or reason.
- Presence is off unless owners allow it.
- Converting an existing session into a shared document strips names and login names from its author records,
  keeping account ids or dropping the author, before anything is signed.
- The documents themselves, and any personal data people type into them, belong to the lab that holds the folder.

## 19. Conformance suite

1. Merge: random operation sets delivered in every order produce the same canonical document.
2. Repair: vectors for each rule.
3. Records: a batch whose `account` differs from its admission; a `wall` or `seen` that goes backwards; an agent `via`
   that tries to grant a role.
4. Trust: a swapped manifest; an unconfirmed issuer; root succession; admission by an owner, by a same-account machine,
   and by owner vouching; certificate expiry after admission.
5. Membership: removals with late batches, `device.accept`, mutual owner removals, the last owner, concurrent downgrades.
6. Forks and compromise: two batches under one number, including one arriving as a sync tool's conflicted copy; a
   compromise notice followed by `device.distrust`.
7. File-system simulator: delayed listings, missing predecessors, duplicates, conflicted copies, temporary and partial
   files, case changes, names whose hash does not match, oversized files.
8. Two implementations, or two versions of one, produce byte-identical snapshots from the same batches.
9. Release checks on real shares: SMB from Windows and macOS, NFS from Linux, and one synced-folder tool.

## 20. Open questions

- Cadence, cap, poll and retention values after measuring on real shares.
- How owners are prompted to review set-aside batches without alert fatigue.
- Repair rules beyond version 1, for product-specific constraints.

## 21. Legal status

- A Labsheaf signature shows which admitted machine, under which account, wrote a change. It is not an electronic
  signature in the sense of 21 CFR Part 11, the EU eIDAS Regulation, Japan's Act on Electronic Signatures and
  Certification Business, or any similar law, and it does not express anyone's approval or intent.
- Issuer certificates are not qualified certificates, and an issuer is not an accredited certification service under
  those laws.
- This specification makes no claim of compliance with any regulation, standard or certification. Anyone who uses
  Labsheaf in a regulated setting remains responsible for meeting the rules that apply to them.
- The specification is provided as is, without warranty of any kind, under the MIT licence in `LICENSE`.

