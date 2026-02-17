# WP-2026-01 — Git Repository Identifier

### Stream44 Studio Workshop Pattern

Authors: Christoph Dorn<br/>
Date: February 15, 2026<br/>
Status: DRAFT — Work in Progress

---

## Introduction

`git` repositories are abitrary containers with the ability to cryptographically sign commits. The signatures ensure that the commit came from a specific source and can include author information in the form of name and email as typically configure in git config.

Having a list of signed commits is great but it does not allow us to uniquely identify a repository. We could simply use the first commit but without additional entropy the hash may not be unique. The **Git Repository Identifier** pattern establishes a convention for a special signed commit, the hash of which is then used to identify the repository.


## Prior Work

There is some exiting work around identifying git repositories with an inception commit at [Open Integrity](https://github.com/OpenIntegrityProject/core). Our pattern is inspired by this prior work with an important changed: The **inception commit** does not need to be the **first commit**.

We found there is a requirement for flexibility around the inception commit:
 * While it is nice to begin every repository with an inception commit, the approach does not work for existing repositories which is a requirement for us.
 * Requiring the inception commit to be the first commit also restricts the evolution of the repository. It requires everything to tie back to the first commit even when the repository and tooling evolve. This is a cripling limitation that was already experienced during development of the Git Repository Identifier logic.


## Perspective

We see the **Git Repository Identifier** as an optional additional layer on top of the git commits that may be added at any point in time. Once the identifier is added, the repository may be validated from that point going backwards and forwards.

It treats the identifier as a stable marker instead of a required starting point. This approach is sufficient to overlay commit policies and enhance the repository with signed trust. Any gaps due to unsigned commits can be specifically ignored and policies can be enforced going forward.

It allows a repository owner to modify the identifier if needed yet still validate continuity from the previous identifier.


## Approach

A **repository identifier commit** is added to a repository as the first commit, making it an **inception commit**, or at any point later making it an **identifier tag commit**.

The commit hash becomes the repository identifier in the form of `did:repo:<hash>`.

### Step 1: Create the Identifier Commit

A signed, empty commit is created using the signing key. The commit carries a `[RepositoryIdentifier]` message prefix and uses `--allow-empty` to ensure no file changes are included in the commit tree. The committer name is set to the key fingerprint to bind the signing key identity to the commit metadata.

```bash
FINGERPRINT=$(ssh-keygen -E sha256 -lf "$SIGNING_KEY" | awk '{print $2}')
DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

GIT_AUTHOR_NAME="$AUTHOR_NAME" \
GIT_AUTHOR_EMAIL="$AUTHOR_EMAIL" \
GIT_COMMITTER_NAME="$FINGERPRINT" \
GIT_COMMITTER_EMAIL="$AUTHOR_EMAIL" \
GIT_AUTHOR_DATE="$DATE" \
GIT_COMMITTER_DATE="$DATE" \
git -c gpg.format=ssh -c user.signingkey="$SIGNING_KEY" \
    commit --allow-empty --no-edit --gpg-sign \
    -m "[RepositoryIdentifier] Establish signed repository identifier." \
    -m "Signed-off-by: $AUTHOR_NAME <$AUTHOR_EMAIL>"
```

### Step 2: Track the Identifier

A second commit writes the DID to a `.repo-identifier` file for fast O(1) lookup of the current identifier. This commit **must use `--signoff`** to add a `Signed-off-by` trailer, and **must use the same signing key, author name and email** as the identifier commit to ensure author consistency:

```bash
COMMIT_HASH=$(git rev-parse HEAD)
echo "did:repo:$COMMIT_HASH" > .repo-identifier
git add .repo-identifier

GIT_AUTHOR_NAME="$AUTHOR_NAME" \
GIT_AUTHOR_EMAIL="$AUTHOR_EMAIL" \
GIT_COMMITTER_NAME="$AUTHOR_NAME" \
GIT_COMMITTER_EMAIL="$AUTHOR_EMAIL" \
git -c gpg.format=ssh -c user.signingkey="$SIGNING_KEY" \
    commit --gpg-sign --signoff \
    -m "[RepositoryIdentifier] Track ${COMMIT_HASH:0:8}"
```

Both commits include a `Signed-off-by` trailer binding the author identity to the commit:
- **Identifier commit**: Uses `-m "Signed-off-by: ..."` as a second message paragraph
- **File commit**: Uses `--signoff` flag which automatically adds the trailer

### Retrieving the Current Identifier

The current identifier is read directly from the file — no git log traversal required:

```bash
cat .repo-identifier
# Output: did:repo:<hash>
```

### Listing All Identifiers

All historical identifiers are found by querying commits that modified `.repo-identifier`. This is O(n) where n is the number of identifier changes, not the total number of commits:

```bash
git log --format=%H --follow -- .repo-identifier
```

For each returned commit, the identifier at that point in time can be read:

```bash
git show <commit>:.repo-identifier
```


## Identifier Validation

Validation involves reading the `.repo-identifier` file, parsing out the commit hash of the **repository identifier commit**, and validating four properties:

1. **Signed** — The identifier commit contains a `gpgsig` field (cryptographic signature).
2. **Empty** — The identifier commit introduces no file changes (its tree matches its parent's tree, or is the empty tree for root commits).
3. **Author Match** — The identifier commit and the file commit that tracks it have the same author name and email.
4. **Key Match** — Both commits are signed with the same signing key (verified via key fingerprint).

```bash
# Read the identifier
DID=$(cat .repo-identifier)
COMMIT_HASH=${DID#did:repo:}

# Find the file commit that tracks this identifier
FILE_COMMIT=$(git log --format=%H -1 -- .repo-identifier)

# Check for signature
HAS_SIG=$(git cat-file -p "$COMMIT_HASH" | grep -c "^gpgsig ")

# Check for empty tree
TREE=$(git log --format=%T -1 "$COMMIT_HASH")
PARENT=$(git log --format=%P -1 "$COMMIT_HASH")

if [ -z "$PARENT" ]; then
    # Root commit: compare against the well-known empty tree hash
    IS_EMPTY=$([ "$TREE" = "4b825dc642cb6eb9a060e54bf8d69288fbee4904" ] && echo 1 || echo 0)
else
    # Non-root: compare tree with parent's tree
    PARENT_TREE=$(git log --format=%T -1 "$PARENT")
    IS_EMPTY=$([ "$TREE" = "$PARENT_TREE" ] && echo 1 || echo 0)
fi

# Check author consistency between identifier commit and file commit
ID_AUTHOR=$(git log --format="%an|%ae" -1 "$COMMIT_HASH")
FILE_AUTHOR=$(git log --format="%an|%ae" -1 "$FILE_COMMIT")
AUTHOR_MATCH=$([ "$ID_AUTHOR" = "$FILE_AUTHOR" ] && echo 1 || echo 0)

# Check signing key consistency - both commits must be signed with the same key
ID_KEY=$(git log --format=%GK -1 "$COMMIT_HASH")
FILE_KEY=$(git log --format=%GK -1 "$FILE_COMMIT")
KEY_MATCH=$([ -n "$ID_KEY" ] && [ "$ID_KEY" = "$FILE_KEY" ] && echo 1 || echo 0)

if [ "$HAS_SIG" -gt 0 ] && [ "$IS_EMPTY" -eq 1 ] && [ "$AUTHOR_MATCH" -eq 1 ] && [ "$KEY_MATCH" -eq 1 ]; then
    echo "VALID: $DID"
else
    echo "INVALID: signed=$HAS_SIG empty=$IS_EMPTY author_match=$AUTHOR_MATCH key_match=$KEY_MATCH"
fi
```


## Use-cases for the Repository Identifier Commit

The identifier marks a stable point in time in the repository signed with a specific key. The same key may be used to sign git commits or construct additional documents such as an XID Document. The order of these events do not matter as **trust** is established by evaluating the commit activity against a policy that specifies certain requirements. These requirements may be adjusted to what is possible or desired for any specific repository.

See [WP-2026-02-GitRepository-Integrity](./WP-2026-02-GitRepository-Integrity.md) for one example of a passive integrity pattern build on this identifier and [WP-2026-03-GordianOpenIntegrity](./WP-2026-03-GordianOpenIntegrity.md) for an active pattern able to rotate signing keys with complete provenance.

