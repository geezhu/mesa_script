## GitLab → GitHub Mirroring Workflow: Handling Broken Annotated Tags

### Language

**English**|**[中文](README.zh-CN.md)**


## 0. Summary: What the workflow changes (and what it doesn’t)

### What it does **not** change

* It **does not modify the upstream GitLab repository**. The workflow runs `git clone --mirror` and operates on a temporary local mirror at `/tmp/src.git`. All fixes happen only inside this temporary repo.

### What it **does** change

Before pushing to the GitHub target repository, the workflow performs a required compatibility fix inside `/tmp/src.git`:

1. **Downgrade known-broken annotated tags to lightweight tags (Plan A)**

   * Before: `refs/tags/<TAG>` → points to a **commit** (type: `commit`) with mail address
   * After:  `refs/tags/<TAG>` → directly points to the **commit** referenced by `object <SHA>` inside that tag object
   * Goal: prevent GitHub’s receive-side validation (`fsck/index-pack`) from rejecting pushes due to malformed tag objects.

---

## 1) Actual Data: Inspecting a broken tag object

Command (example `feb356...`):

```bash
git cat-file -p feb356caff82e996ba0b898c02383fdfa3effc5f
```

Expected structure (example, fields may vary):

```
object 20330f422f30e873c33b076e4f584ebb4544d733
type commit
tag R300_DRIVER_0
tagger Nicolai Haehnle <prefect_@gmx.net>
```

The key is the `tagger` line. A valid annotated tag must include:

```
tagger Name <email> <unix_timestamp> <timezone>
```

If the content ends right after `>` (newline immediately) or the timestamp/timezone formatting is invalid (e.g., missing required spaces), the tag object is malformed and will be rejected by GitHub during push.

---

## 2) Generating the broken SHA list (`bad_tag_objects.txt`)

Recommended command (strips the trailing `:`):

```bash
git fsck --full --strict 2>&1 \
| awk '/^error in tag [0-9a-f]{40}:/ {sha=$4; sub(/:$/,"",sha); print sha}' \
| sort -u > bad_tag_objects.txt
```

Example `bad_tag_objects.txt`:

```
feb356caff82e996ba0b898c02383fdfa3effc5f
5562545645baea8e3d8409d712ab0a5da90e355c
...
```

---

## 3) Prerequisite: Build a mapping (tag object SHA → tag name)

Run in the temporary mirror repo (only annotated tags):

```bash
git for-each-ref refs/tags --format='%(objectname) %(objecttype) %(refname:short)' \
  | awk '$2=="tag"{print $1, $3}' > /tmp/tagobj_to_name.txt
```

---

## 4) Plan A (current): Downgrade broken annotated tags to lightweight tags

### Core idea

For each broken tag object SHA:

1. Resolve its tag name
2. Read `object <sha>` from the tag object (the real target commit)
3. Update `refs/tags/<TAG>` to point directly to that commit (lightweight tag)

### Core code (run before pushing tags)

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  target="$(git cat-file -p "$tagobj" | awk '/^object /{print $2; exit}')"
  git update-ref "refs/tags/$tag" "$target"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 5) Plan B: Skip broken tags (more conservative)

### Core idea

Do not repair tags; instead, delete those tag refs in the temporary mirror so they won’t be pushed.

### Core code (delete tags in the temporary repo)

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  git tag -d "$tag"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 6) Plan C: Recreate as valid annotated tags (closer to upstream, but more complex)

### Core idea

Recreate a new, valid annotated tag (new tag object) and overwrite the old tag name.

### Core code

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  target="$(git cat-file -p "$tagobj" | awk '/^object /{print $2; exit}')"
  git tag -a -f "$tag" "$target" -m "recreated tag for mirror"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 7) FAQ: Does Plan A require force push?

* If the target repository already has the same tag name (common), updating it usually requires **force**.
* Using `+refs/tags/...` in your push (or `--force`) works for Plans A/B/C.
