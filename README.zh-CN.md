# GitLab → GitHub 同步脚本：坏 Annotated Tag 处理说明

## 0. 结论：脚本改了什么、没改什么

### 没改什么

* **不会改动 GitLab 源仓库**。从 GitLab `git clone --mirror` 得到的是本地临时仓库 `/tmp/src.git`，所有处理都发生在这个临时仓库里。

### 改了什么

同步脚本在 **推送到 GitHub 目标仓库之前**，会对本地镜像 `/tmp/src.git` 做“为了能成功同步而必须的修改”。目前主要是：

1. **把已知“坏的 annotated tag”降级为 lightweight tag（方案 A）**

* 原状态：`refs/tags/<TAG>` → 指向一个 **commit**
* 处理后：`refs/tags/<TAG>` → 直接指向该 tag object 中 `object <SHA>` 对应的 **commit**
* 目的：避免 GitHub 接收端在 `fsck/index-pack` 校验时，因为 tag object 不合法而拒收推送。

---

## 1) Actual Data：如何查看某个坏 tag object 的真实内容

命令（以 `feb356...` 为例）：

```bash
git cat-file -p feb356caff82e996ba0b898c02383fdfa3effc5f
```

你会看到类似结构（示例，不保证完全一致）：

```
object 20330f422f30e873c33b076e4f584ebb4544d733
type commit
tag R300_DRIVER_0
tagger Nicolai Haehnle <prefect_@gmx.net>
```

判断是否“坏”的关键在 `tagger` 行。正常格式必须是：

```
tagger Name <email> <unix_timestamp> <timezone>
```

如果 `>` 后面直接换行、或时间戳/时区之间缺少空格，就属于不合法对象，推送到 GitHub 时会被拒绝。

---

## 2) 获取损坏的 SHA 列表（bad_tag_objects.txt）

建议用下面命令生成（自动去掉末尾冒号）：

```bash
git fsck --full --strict 2>&1 \
| awk '/^error in tag [0-9a-f]{40}:/ {sha=$4; sub(/:$/,"",sha); print sha}' \
| sort -u > bad_tag_objects.txt
```

最终 `bad_tag_objects.txt` 的内容示例为：

```
feb356caff82e996ba0b898c02383fdfa3effc5f
5562545645baea8e3d8409d712ab0a5da90e355c
...
```

---

## 3) 修复前置：建立映射（tag object SHA → tag 名）

在临时镜像仓库中建立映射（只取 annotated tags）：

```bash
git for-each-ref refs/tags --format='%(objectname) %(objecttype) %(refname:short)' \
  | awk '$2=="tag"{print $1, $3}' > /tmp/tagobj_to_name.txt
```

---

## 4) 方案 A：降级坏 annotated tag 为 lightweight tag（当前使用）

### 核心思路

对每个坏 tag object SHA：

1. 查出它对应的 tag 名
2. 从 tag object 里读出 `object <sha>`（真正指向的 commit）
3. 直接把 `refs/tags/<TAG>` 改成指向该 commit（变为 lightweight tag）

### 核心代码（推送 tags 之前执行）

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  target="$(git cat-file -p "$tagobj" | awk '/^object /{print $2; exit}')"
  git update-ref "refs/tags/$tag" "$target"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 5) 方案 B：跳过坏 tag（更保守）

### 核心思路

不去修复 tag，而是在临时镜像仓库里直接删掉这些 tag 引用，这样推送时不会再包含它们。

### 核心代码（临时仓库中删除）

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  git tag -d "$tag"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 6) 方案 C：重建为合法 annotated tag（更像原仓库，但更复杂）

### 核心思路

对同名 tag 重新创建一个合法的 annotated tag（会生成新的 tag object），并覆盖原 tag。

### 核心代码

```bash
while read -r tagobj; do
  tag="$(awk -v s="$tagobj" '$1==s{print $2; exit}' /tmp/tagobj_to_name.txt)"
  target="$(git cat-file -p "$tagobj" | awk '/^object /{print $2; exit}')"
  git tag -a -f "$tag" "$target" -m "recreated tag for mirror"
done < "$GITHUB_WORKSPACE/bad_tag_objects.txt"
```

---

## 7) FAQ：方案 A 是否需要强制推送？

* 如果目标仓库已存在同名 tag（通常会发生），更新 tag ref 一般需要 **force**。
* 你推送 tags 时使用的 `+refs/tags/...`（或等价 `--force`）即可复用到 A/B/C 三个方案。
