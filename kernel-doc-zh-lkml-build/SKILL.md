---
name: kernel-doc-zh-lkml-build
description: Find a Linux kernel documentation patch or series Message-ID by subject in the user's HUST mail archive with hust-mail-cli, retrieve it with b4, and compare full documentation builds before and after applying it in kernel-doc-zh. Use when the input is a patch title or cover letter title.
---

# LKML 补丁文档编译对比

输入为单个 patch 的标题或 patchset cover letter 的标题。目标是通过 `hust-mail-cli` 在用户的邮件归档中找到**准确的邮件版本和 Message-ID**，再用 `b4` 取回用户指定范围的补丁，并在 `kernel-doc-zh` 的完整隔离副本中比较补丁前后的文档构建。只凭相同的 warning/error 数量不能判定没有回归。

## 1. 核对本地规则和目标树

- 找到用户指定的仓库；常见位置是 `~/kernel-doc-zh` 或 `~/Repos/kernel-doc-zh`。以实际 `git rev-parse --show-toplevel` 为准，不要把路径当作固定值。
- 先读该树的 `Documentation/translations/zh_CN/how-to.rst`，尤其是“进行第一次文档编译”和“编译和检查”。以当前文件为准。该指南要求每次构建先 `make cleandocs`，再 `make htmldocs`，并检查 `Documentation/output/` 中的页面。指南示例使用 `sphinx_latest` 虚拟环境；可以使用已有、满足 `Documentation/sphinx/requirements.txt` 的等效环境，前后两次必须相同。
- 记录分支、HEAD、`git status --short --branch`、可用磁盘空间和构建工具。不要切换、清理或修改原工作区；未提交内容不进入基线。用完整内核源码的隔离 clone，避免只复制 `Documentation/` 或仅构建 `SPHINXDIRS=translations/zh_CN`。`/tmp` 空间不足时选择有足够容量且可写的磁盘目录。

## 2. 通过 hust-mail-cli 从标题找到确切邮件

1. 使用 `hust-mail-cli search --subject '<标题中的独特短语>' --json` 搜索本地邮件归档。它的 `--subject` 是不区分大小写的子串匹配；标题包含 `[PATCH vN X/Y]`、`Re:` 等前缀时，先用核心短语扩大候选，再按完整标题、作者和日期缩小。可加 `--after`、`--before`、`--from`，但不要过早用 `--limit` 截断候选。若当前环境不允许写归档索引，加 `--no-index`；这会扫描源邮件，可能较慢。
2. 搜索输出为 JSON Lines，邮件文件路径在 `path` 字段。对候选邮件用 `hust-mail-cli read --raw <path>` 读取原始邮件头，核对 `Subject`、`From`、`Date`、`Message-ID`、`In-Reply-To`，以及版本号和补丁序号/总数。只把必要的邮件头和候选摘要写入工作日志，不输出私人正文或附件。移除 Message-ID 外层的 `<` `>` 后交给 b4；记录 `https://lore.kernel.org/r/<Message-ID>` 作为对应的公开归档链接。同名、重发或多版本不能凭搜索排序猜测；依据用户指定版本或现有上下文选择，确实无法确定时列出候选并询问。
3. `hust-mail-cli search` 只检索**已同步到本地**的邮件，不代表完整 LKML。无结果时先检查标题关键词、日期过滤和归档覆盖范围；如需获取尚未同步的邮件，用 `hust-mail-cli sync --days <范围>` 或指定 `--mailbox` 同步适当范围后重试。无法定位时报告本地归档的限制，不编造 Message-ID。
4. 输入为 cover letter 标题时，使用该 cover 的 Message-ID，预期获得整套补丁。输入为单个 patch 标题时，先确认其是否属于 patchset：若用户要**该单个补丁**，使用 `b4 am -P _ <该补丁的Message-ID>`；若要整套，则使用对应 cover 或该补丁的 Message-ID 并核对 b4 选中的范围。单独应用有前置依赖的补丁时，明确记录所需前置补丁，不把套件结果冒充单补丁结果。

## 3. 用 b4 取回并核对范围

- 在隔离工作目录运行 `b4 am -C -o <邮件输出目录> <Message-ID>`，单补丁按上一节加 `-P _`。`-C` 表示不用旧缓存，不是选择 cover。若 b4 默认选到同线程的较新版本而用户指定旧版本，使用 `-v <版本号>` 重新生成；必要时用 `-c` 检查新版本。不要用 `b4 shazam` 操作原仓库。
- 检查 b4 输出的主题、版本、补丁数、顺序、cover、`Base:` 和生成的 `.mbx`，并与 `hust-mail-cli` 找到的邮件核对。若 b4 缓存或状态目录不可写，在隔离目录设置可写的 `XDG_CACHE_HOME` / `XDG_DATA_HOME`。检索失败、补丁缺失或身份不明时停止应用，报告具体限制。
- 若邮件给出 `base-commit`，在隔离 clone 中验证该提交并以它作为**两次构建共同的基线**。若没有，记录所选已提交基线（通常为原仓库 HEAD）、来源和适用性；不能干净应用时，不通过强行修改补丁来制造可比结果。补丁依赖或 base 无法确定时说明限制。

## 4. 在完整隔离树中做前后两次构建

1. 从目标仓库创建完整 clone，检出选定基线，确认隔离树干净。保留原仓库状态记录；clone 和日志放在有足够空间的位置。两次构建用同一 Sphinx 环境、环境变量和完整 `make` 目标。
2. **补丁前**：在隔离树根执行 `make cleandocs`，确认成功后执行 `make htmldocs`。分别保存完整 stdout/stderr、退出码、基线 commit 和重要生成页的副本或清单。不要让 `tee` 掩盖 `make` 的失败：用重定向，或启用 `set -o pipefail` 并记录各命令状态。
3. 在该隔离树中对 b4 生成的 `.mbx` 执行 `git am`。验证应用的提交数和主题；执行 `git diff --check <base>..HEAD`。如果应用失败，保留现场并报告，不能声称完成补丁后构建。
4. **补丁后**：再次严格执行 `make cleandocs`，然后 `make htmldocs`，以与基线相同方式记录日志、退出码和生成页面。`cleandocs` 不能省略，否则增量构建可能漏报真实警告。
5. 比较**完整诊断文本**，而不只是 warning/error 计数：标出新增、消失和变化的消息及其文件/行号；将补丁前已有的问题与新问题区分。检查受影响中文页面在 `Documentation/output/` 的渲染、目录链接和锚点；有新增或改动的源文档时，与英文原文及 `:Original:` 路径核对相关内容。构建失败时报告失败阶段和日志中的首个相关错误，不把未完成的对比写成通过。

## 5. 汇报

给出检索到的标题、版本、Message-ID/lore 链接、b4 取得的补丁范围、基线与补丁后 commit、两次 `cleandocs`/`htmldocs` 退出码，以及新增或消失的准确诊断和页面问题。注明构建环境或补丁 base 的限制，并确认原仓库的工作区状态未被改变。只有两次完整构建及诊断比较确实完成，才给出文档编译回归结论。
