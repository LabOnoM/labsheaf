# Labsheaf

Labsheaf is an open protocol for working together on structured analysis documents, such as a pipeline drawn as a node
graph with its notes and review threads, through a folder that every collaborator can reach: a lab NAS over SMB or NFS,
or a synced folder such as Syncthing, OneDrive, Dropbox, Nextcloud or Git.

There is no server in the path. Each machine keeps a full local copy, writes its own signed, write-once change files into
the shared folder, and merges everyone's changes the same way, so collaboration keeps working offline and the storage
stays under the lab's own control.

## Properties

- **Convergence:** every machine that has accepted the same changes computes the same document, byte for byte.
- **Authorship and integrity:** every change is signed by an admitted machine of its author's account, and anyone can
  check this offline.
- **Independence:** once a machine is admitted to a document, its changes stay valid whatever happens to licences,
  certificates or the issuer.
- **Openness:** documents are plain JSON, readable without any particular software.
- **Real storage:** designed for shared file systems and sync tools whose files arrive late, out of order or twice.

Labsheaf does not encrypt document contents: who can read a document is decided by the folder's own access control.

## Legal notes

A Labsheaf signature shows which admitted machine, under which account, wrote a change. It is not an electronic
signature under 21 CFR Part 11, eIDAS, Japan's Act on Electronic Signatures and Certification Business or similar laws,
and the protocol makes no compliance claim. See section 21 of [`SPEC.md`](SPEC.md).

## Status

Draft 0.2. The specification is in [`SPEC.md`](SPEC.md). Formats may change before version 1.0. A conformance suite and
a reference implementation will be published here.

## Contributing

Questions, reviews and proposals are welcome as issues. Pull requests are accepted under the terms in
[`CONTRIBUTING.md`](CONTRIBUTING.md). Please report security problems privately, as described in
[`SECURITY.md`](SECURITY.md), rather than in a public issue.

## Licence

MIT, copyright Nexitia LLC. See [`LICENSE`](LICENSE).
