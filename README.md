# claude-merge-sessions

用 Claude 桌面 App（Code 标签页）时，如果你有好几个账号，切换账号以后左边的会话栏会变成另一套。这个小工具把几个账号的本地会话列表合并成一份，切到哪个账号看到的都一样。

> ⚠️ 这是非官方的社区工具，跟 Anthropic 没有关系。它依赖桌面 App 内部的存储格式，App 更新后可能会失效。每次改动前它都会自动备份，可以一键撤销。

[English](#english)

## 为什么会这样

桌面 App 把侧边栏的会话索引按「账号/组织」分开存放：

```
~/Library/Application Support/Claude/claude-code-sessions/<账号UUID>/<组织UUID>/local_<会话ID>.json
```

但对话内容本身（`~/.claude/projects/**/<cliSessionId>.jsonl`）是所有账号共用的。所以只需要在各账号目录之间把这些 `local_*.json` 补齐、并且以最新的版本为准，切换账号后侧边栏就是同一份了，点开的会话也能接着聊。

## 安装

要求：macOS，Python 3.9 或更高版本（系统自带的 `/usr/bin/python3` 就够用），不需要装任何第三方库。

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/darkfulqy/claude-merge-sessions/main/claude-merge-sessions -o ~/.local/bin/claude-merge-sessions
chmod +x ~/.local/bin/claude-merge-sessions
```

记得确认 `~/.local/bin` 已经在你的 `PATH` 里。

## 用法

**每次切换账号之前跑一下：**

```bash
claude-merge-sessions
```

| 命令 | 作用 |
| --- | --- |
| `claude-merge-sessions` | 合并（改动前会先备份） |
| `claude-merge-sessions --dry-run` | 只预览会改什么，不动任何文件 |
| `claude-merge-sessions -v` | 同时列出每个新增、更新、移除的会话标题 |
| `claude-merge-sessions --undo` | 撤销最近一次合并 |
| `claude-merge-sessions --undo 20261111-013801` | 撤销指定的那一次（名字就是备份目录名） |
| `claude-merge-sessions --include-active` | App 开着也写入当前登录账号（不推荐） |

切换账号后如果侧边栏没有刷新，用 ⌘Q 彻底退出 App 再打开就好。

输出示例：

```
找到 2 个账号/组织目录：
  aaaaaaaa…/oooooooo…  会话 384 个（归档 117）  ← 当前登录账号（App 运行中，本次不改它）
  bbbbbbbb…/pppppppp…  会话 257 个（归档 161）
合并后共 545 个会话（已删除 5 个）。

执行：
  aaaaaaaa…/oooooooo…  暂缓 165 项（等切换账号后再跑一次，或退出 App 后跑）
  bbbbbbbb…/pppppppp…  新增 288，更新 96，删除标记 1，重建归档索引

改动前的文件已备份到：~/.claude-session-merge/backups/20261111-013801
```

## 合并规则

- 同一个会话在多个账号里都有时，以**文件修改时间最新**的那份为准；时间一样就再比较 `lastActivityAt`。
- 在某个账号里**删掉**的会话（`deleted_<id>` 删除标记比所有副本都新），在其它账号里也会移除，移除前先备份。如果删除标记反而比某份会话旧，说明之后又用过它，那就保留会话、撤掉删除标记。
- 归档、星标/置顶、标题都写在会话文件里，会跟着一起同步。`archived-sessions.idx` 会按合并后的结果重建。
- 会处理 `claude-code-sessions` 下面所有的「账号/组织」目录，包括还没有任何会话的空目录。

## 安全措施

- **App 开着的时候，默认不改当前登录账号的目录**，免得跟 App 内存里的状态冲突，只把内容补到其它账号。所以最省心的用法就是「切换前跑一下」；如果先 ⌘Q 退出 App 再跑，就是完全的双向合并。
- 写文件用「临时文件 + 原子重命名」，不会留下写了一半的文件。正在被写、暂时读不出来的会话文件，这一次会直接跳过。
- 每次改动前都会把要覆盖或移走的文件备份到 `~/.claude-session-merge/backups/<时间>/`，并记一份 `manifest.json`，`--undo` 就是按它来还原的。
- 重复运行是安全的：已经一致的话，什么都不会改。

## 不会合并的内容

- 定时任务（`scheduled-tasks.json`），免得两个账号把同一个任务各跑一遍。
- `backlog`，以及侧边栏的分组、排序这类显示偏好。
- 云端会话。这个工具只处理本机的 `local_` 会话。

## English

**claude-merge-sessions** keeps the Code-tab session sidebar of the Claude desktop app identical across multiple accounts on one Mac.

The desktop app keeps its sidebar index per account/org under `~/Library/Application Support/Claude/claude-code-sessions/<account>/<org>/local_<id>.json`, while the transcripts in `~/.claude/projects` are shared. This script unions those index files across all account/org folders. On conflict the newest file wins, deletions (`deleted_<id>` tombstones) propagate, and the archive index gets rebuilt. While the app is running it never writes to the signed-in account's folder, so run it right before you switch accounts. Every change is backed up and can be reverted with `--undo`.

Unofficial and not affiliated with Anthropic. It relies on internal, undocumented file formats that may change. macOS, Python ≥ 3.9, no dependencies.

## License

[MIT](LICENSE)
