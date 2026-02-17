# WP-2026-02 — Git Repository Integrity

### Stream44 Studio Workshop Pattern

Authors: Christoph Dorn<br/>
Date: February 16, 2026<br/>
Status: DRAFT — Work in Progress

---

## Introduction

`git` repositories are abitrary containers with the ability to cryptographically sign commits. The signatures ensure that the commit came from a specific source and can include author information in the form of name and email as typically configure in git config.

As a baseline, one can ensure all commits to a repository are signed using a key and validate that the author name and email did not change for the same key. This provides a mechanism to verify the origin of commits but there is no additional information regarding whether the keys used or authors specified are allowed to commit changes.

To conduct additional validation we require two things:
- A trust root that establishes a point in time from which policies apply
- A mechanism to cryptographically define policies and policy changes

Each of these layers makes additional validation possible and this pattern aims to provide a cohesive validation approach able to establis the integrity of a repository in terms of trust and provenance.


## Overview

This **Git Repository Integrity** pattern is designed to validate various aspects of a git repository in terms of trust and provenance.

The following aspects are validated:

- Commit Origin
  - Were commits signed off with author details?
  - Were commits signed with private keys?
  - Do author details match across commits from the same key?
- Commit Trust
  - Is each author allowed to commit?
- Trust Provenance
  - Is each author commit permission recorded and authorized by a trusted party.

The approach works for *plain* `git` repositories as well as repositories enhanced with [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md) and [WP-2026-03-GitRepository-GordianOpenIntegrity](./WP-2026-03-GitRepository-GordianOpenIntegrity.md). The latter two are required for enhanced trust and provenance validation.


## Prior Work

There is some exiting work around adding integrity rules to git repositories with a `.repo/` directory to store decisions (signed config files). More details here: https://github.com/OpenIntegrityProject/core/blob/main/docs/Open_Integrity_Repo_Directory_Structure.md

We believe it is important to maintain flexibility in every aspect when defining such rules and enforcing them with tooling. We also see the same concept of defining rules and decisions in a cryptographically rigerous way useful for many other things.

We have thus specified a new pattern at [WP-2026-03-GitRepository-GordianOpenIntegrity](./WP-2026-03-GitRepository-GordianOpenIntegrity.md) that implements a generic decision layer with cryptographic provenance using XID Documents tied to a [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md).

Once a **Gordian Open Integrity** document has been established in a repository, one can introduce a new policy document to implement the same featureset as proposed for the `.repo/` approach.


## Validations

Validation is organized into three progressive layers, each building on the previous:

| Layer | Name | Requirements | What it validates |
|-------|------|-------------|-------------------|
| **1** | Commit Origin | `git`, `ssh-keygen` | Signatures, sign-offs, author consistency |
| **2** | Repository Identifier | Layer 1 + `.repo-identifier` | Identifier commit validity, DID binding |
| **3** | Gordian Open Integrity | Layer 2 + Gordian stack | Provenance chains, XID stability, envelope assertions, document registry |
| **4** | XID Document Governance | Layer 3 + XID document | Signer authorization, repository identifier binding |

### Layer 1 — Commit Origin

Layer 1 validates git-level properties of commits. It requires only `git` and `ssh-keygen` — no trust policy or Gordian stack is needed.

#### 1.1 Commit Signatures

Validates that every commit in the repository is cryptographically signed.

**Without allowed signers** — checks for the presence of a `gpgsig` field in each commit's raw object:

```bash
git cat-file -p <hash>
# Valid if output contains: gpgsig -----BEGIN SSH SIGNATURE-----
```

**With allowed signers** — verifies each signature against the known public keys:

```bash
# Write allowed signers to temporary file
echo '<email> namespaces="git" <public-key>' > /tmp/allowed-signers

# Verify each commit
git -c gpg.ssh.allowedSignersFile=/tmp/allowed-signers verify-commit <hash>
# Valid if output contains: Good "git" signature
```

| Check | Condition |
|-------|-----------|
| Commit signed (basic) | Raw commit object contains `gpgsig ` field |
| Signature valid (with signers) | `git verify-commit` output contains `Good "git" signature` |

Result fields: `totalCommits`, `signedCommits`, `unsignedCommits`, per-commit `hash`, `message`, `signatureValid`, `keyFingerprint`, `authorName`, `authorEmail`.

#### 1.2 Commit Sign-offs

Validates that every commit includes a `Signed-off-by:` trailer, binding the author identity to the commit message.

```bash
git log --format=%B -1 <hash>
# Valid if body contains: Signed-off-by:
```

| Check | Condition |
|-------|-----------|
| Sign-off present | Commit message body contains `Signed-off-by:` |

Result fields: `totalCommits`, `signedOffCommits`.

#### 1.3 Author Consistency

Validates that each signing key is associated with exactly one author identity (name + email). If a key is used by multiple author identities, this indicates either a misconfiguration or an impersonation attempt.

```bash
git log --format='%GK|%an|%ae' --reverse
# Group by key fingerprint, check for multiple author identities per key
```

| Check | Condition |
|-------|-----------|
| Author consistent | Each signing key fingerprint maps to exactly one `<name> <<email>>` pair |

Result fields: `totalCommits`, `authors` (map of key fingerprint → list of author identities).

### Layer 2 — Repository Identifier

Layer 2 validates the repository identifier established by [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md). It requires a `.repo-identifier` file in the repository.

#### 2.1 Identifier Validation

Reads the `.repo-identifier` file, parses the `did:repo:<hash>` value, and validates four properties of the identifier commit:

```bash
DID=$(cat .repo-identifier)
COMMIT_HASH=${DID#did:repo:}
FILE_COMMIT=$(git log --format=%H -1 -- .repo-identifier)
```

| Check | Condition |
|-------|-----------|
| **Signed** | Identifier commit raw object contains `gpgsig ` field |
| **Empty** | Identifier commit tree matches parent tree, or equals the empty tree hash `4b825dc642cb6eb9a060e54bf8d69288fbee4904` for root commits |
| **Author match** | Identifier commit and file commit share the same author name and email |
| **Key match** | Both commits are signed with the same SSH key (verified via signature block comparison) |

Result fields: `valid`, `commitHash`, `did`, `isSigned`, `isEmpty`, `authorMatch`, `keyMatch`, `keyFingerprint`, `authorName`, `authorEmail`.

### Layer 3 — Gordian Open Integrity

Layer 3 validates the [Gordian Open Integrity](./WP-2026-03-GitRepository-GordianOpenIntegrity.md) system. It requires the Gordian cryptographic stack (XID Documents, Gordian Envelopes, Provenance Marks) and operates on the provenance history collected from git commits that modified the `.o/GordianOpenIntegrity.yaml` file.

#### 3.1 Repository Verification

Verifies the inception provenance chain and all commit signatures. This is the primary entry point for verifying a repository's integrity given only a cloned repository and an optional published provenance mark.

**Steps:**

1. **Collect provenance history** — Walk all commits that modified `.o/GordianOpenIntegrity.yaml`, parse each version's Gordian Envelope, extract the XID, SSH keys, and provenance mark.

2. **Verify provenance mark monotonicity** — Confirm that the sequence number of each provenance mark is strictly greater than the previous:

    | Check | Condition |
    |-------|-----------|
    | Marks monotonic | `marks[i].seq() > marks[i-1].seq()` for all consecutive pairs |

3. **Verify published mark** — If a published mark identifier is provided, confirm it matches the latest provenance mark:

    | Check | Condition |
    |-------|-----------|
    | Mark matches latest | Published mark identifier equals the identifier of the last provenance mark in the chain |

4. **Collect allowed signers** — Extract all SSH public keys from all provenance versions' `GordianOpenIntegrity.SigningKey` assertions. These form the allowed signers list.

5. **Audit commit signatures** — Verify every commit in the repository against the collected allowed signers list (delegated to Layer 1 signature verification with allowed signers).

    | Check | Condition |
    |-------|-----------|
    | All signatures valid | `invalidSignatures === 0` |

6. **Verify XID stability** — Confirm the XID identifier is the same across all provenance versions:

    | Check | Condition |
    |-------|-----------|
    | XID stable | All provenance versions report the same XID value |

Result fields: `valid`, `xid`, `did`, `marksMonotonic`, `markMatchesLatest`, `xidStable`, `totalCommits`, `validSignatures`, `invalidSignatures`, `provenanceVersions`, `issues[]`.

#### 3.2 Document Verification

Verifies a specific decision document registered in the inception envelope. Extends repository verification with document-specific checks.

**Steps 1–3** are identical to repository verification but applied to the document's provenance history (commits that modified the document's YAML file).

**Additional steps:**

4. **Verify document self-reference** — The latest document envelope must contain a `GordianOpenIntegrity.Document` assertion matching the document's file path:

    | Check | Condition |
    |-------|-----------|
    | Document path valid | `GordianOpenIntegrity.Document` assertion value equals the expected document path |

5. **Verify documents map** — The latest inception envelope must contain a `GordianOpenIntegrity.Documents` assertion (JSON map) that maps the document path to the document's XID:

    | Check | Condition |
    |-------|-----------|
    | Documents map valid | `JSON.parse(assertion)[documentPath] === documentXid` |

6. **Verify XID stability** — Same as repository verification, applied to the document's provenance history.

7. **Audit commit signatures** — Collects SSH keys from both the document and inception provenance histories, then audits all repository commits against the combined allowed signers list.

Result fields: `valid`, `xid`, `documentPathValid`, `documentsMapValid`, `marksMonotonic`, `markMatchesLatest`, `xidStable`, `totalCommits`, `validSignatures`, `invalidSignatures`, `provenanceVersions`, `issues[]`.

### Layer 4 — XID Document Governance

Layer 4 validates changes in the XID document as they relate to the git repository. It ensures that signing key changes are properly authorized in the XID document and that the repository identifier is correctly bound.

#### 4.1 Signer Authorization

Validates that every commit in the repository was signed by a key that appears in at least one version of the XID document's `GordianOpenIntegrity.SigningKey` assertion. This ensures no unauthorized keys were used to sign commits.

| Check | Condition |
|-------|----------|
| All signers authorized | Every commit's signature verifies against a key from the provenance history |

Result fields: `valid`, `signersAllAuthorized`, `authorizedKeyCount`, `issues[]`.

#### 4.2 Repository Identifier Binding

Validates that the `GordianOpenIntegrity.RepositoryIdentifier` assertion in the inception envelope points to the actual first (inception) commit of the repository.

```bash
# Get the first commit
FIRST_COMMIT=$(git rev-list --max-parents=0 HEAD)
# Compare with the DID from the inception envelope
assert IDENTIFIER_HASH == FIRST_COMMIT
```

| Check | Condition |
|-------|----------|
| Identifier is inception | `did:repo:<hash>` from inception envelope matches the repository's root commit |

Result fields: `valid`, `repoIdentifierIsInceptionCommit`, `identifierCommit`, `inceptionCommit`, `issues[]`.


### Strict Mode

All verification functions (`verify`, `verifyDocument`, `validate`) accept an optional `strict` parameter with flags that enable Layer 4 checks:

```
strict: {
    repoIdentifierIsInceptionCommit: true | false,
    signersAllAuthorized: true | false,
}
```

| Flag | Effect |
|------|--------|
| `repoIdentifierIsInceptionCommit` | Enables Layer 4.2 — fails if the repository identifier does not point to the first commit |
| `signersAllAuthorized` | Enables Layer 4.1 — fails if any commit is signed by a key not present in the XID document history |

When strict mode is not specified, only Layers 1–3 are enforced.


### Comprehensive Validation

The `validate` function runs all applicable layer checks and returns a structured report:

```
{
    valid: boolean,
    layers: {
        commitSignatures: { ... },
        commitSignoffs: { ... },
        authorConsistency: { ... },
        repositoryIdentifier: { ... },
        provenanceChain: { ... },
        xidStability: { ... },
        repoIdentifierIsInceptionCommit: { ... },  // strict only
        signerAuthorization: { ... },               // strict only
    },
    totalCommits: number,
    signedCommits: number,
    unsignedCommits: number,
    issues: string[],
}
```

Each layer result is independently accessible for targeted inspection.


## Failure Messages

| Layer | Issue | Cause |
|-------|-------|-------|
| 1 | `Commit <hash> is not signed` | Commit missing `gpgsig` field |
| 1 | `N commit(s) have invalid or unverifiable signatures` | Signatures failed verification against allowed signers |
| 1 | `Commit <hash> missing Signed-off-by trailer` | Commit message lacks sign-off |
| 1 | `Key <fp> used by multiple authors: ...` | Same signing key used with different author identities |
| 2 | `No repository identifier found` | `.repo-identifier` file missing or unreadable |
| 2 | `Invalid .repo-identifier content` | File does not contain a valid `did:repo:` value |
| 2 | `Commit <hash> not found` | Referenced identifier commit does not exist |
| 3 | `No provenance documents found in repository history` | No commits modified the provenance YAML file |
| 3 | `Provenance mark sequence regressed: seq N <= M` | Provenance mark sequence is not strictly monotonic |
| 3 | `Published mark "X" does not match latest provenance mark "Y"` | Published mark does not match the chain head |
| 3 | `N commit(s) have invalid signatures` | Commits not signed by keys declared in provenance |
| 3 | `XID changed across provenance versions` | XID identity is not stable across document versions |
| 3 | `Document envelope missing GordianOpenIntegrity.Document assertion` | Document envelope lacks self-reference |
| 3 | `Document path assertion "X" does not match expected "Y"` | Self-reference points to wrong path |
| 3 | `Documents map XID mismatch for "path"` | Inception documents map has wrong XID for document |
| 3 | `Inception envelope missing GordianOpenIntegrity.Documents assertion` | Inception envelope lacks documents registry |
| 3 | `Failed to parse GordianOpenIntegrity.Documents assertion` | Documents map is not valid JSON |
| 3 | `No inception provenance found in repository` | Inception provenance history is empty |
| 4 | `N commit(s) signed by keys not authorized in XID document` | Commits signed by keys absent from provenance history |
| 4 | `Inception envelope missing GordianOpenIntegrity.RepositoryIdentifier assertion` | No repo ID in inception envelope |
| 4 | `Invalid repository identifier format: X` | Repo ID assertion is not a valid `did:repo:` value |
| 4 | `Repository identifier commit X is not the inception (first) commit Y` | Repo ID does not match root commit |

## Implementation

The reference implementation is the `GitRepositoryIntegrity` capsule at:

```
t44/caps/providers/blockchaincommons.com/GitRepositoryIntegrity
```

All four validation layers, including `verify`, `verifyDocument`, and `validate`, are implemented in this capsule. The `GordianOpenIntegrity` capsule maps `GitRepositoryIntegrity` and delegates all verification to it:

```
t44/caps/providers/blockchaincommons.com/GordianOpenIntegrity
```

See [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md) for the repository identifier pattern and [WP-2026-03-GitRepository-GordianOpenIntegrity](./WP-2026-03-GitRepository-GordianOpenIntegrity.md) for the full Gordian Open Integrity specification.
