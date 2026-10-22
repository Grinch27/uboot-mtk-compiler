# AGENTS.md — U-Boot MTK Compiler

<!--
Repository: Grinch27/uboot-mtk-compiler
Part I: common GitHub Actions standards adapted from Grinch27/openwrt-compiler/AGENTS.md.
Part II: repository-specific instructions. Existing content, if present, is preserved verbatim in Part II-B.
This deployment changes only AGENTS.md; it does not constitute approval for workflow or source modification.
-->

# Part I — GitHub Actions 通用规范（统一维护）

<!-- BEGIN: NEW WORKFLOW RULES -->
<!-- COMMON_RULES_VERSION: 1.0.0 -->
<!-- COMMON_RULES_SOURCE: Grinch27/github-actions/docs/agents/part-i.md -->

## 1. 适用范围与基本原则

本部分用于规范本项目及未来可复用的 GitHub Actions 工作流设计。

### 1.1 基本原则

- GitHub Actions 工作流分为 Non-container 和 Container 两种路线。
- 所有新项目原则上先实现 Non-container 基础路线。
- 基础路线稳定后，才开展以追新为目的的 Container 路线。
- 所有远程 Action 的 `uses` 引用必须采用经过探测的 `main` 或 `master` 分支。
- 所有适用工作流必须使用统一的 `runner-image` Job。
- 其他普通 Ubuntu Runner Job 必须使用 `needs.runner-image.outputs.RUNNER_IMAGE`。
- 工作流语法必须符合 GitHub 官方文档。
- 优先使用 `Grinch27/github-actions` 中已有的共享 Actions。
- 不因追求简化、统一或追新而破坏已经验证的构建功能。
- 不擅自修改已有的权限、私有数据处理、构建或发布边界。

### 1.2 规范层级

本文件分为两部分：

- Part I：通用 GitHub Actions 工作流规范。
- Part II：各仓库独立维护的项目专属约束（不参与同步）。

两部分相互独立，但在实际执行时共同生效。

通用工作流规范不能覆盖项目专属的安全、隐私、设备操作和授权限制。

如果两部分存在实质冲突，应先报告并确认，不得自行忽略其中一部分。

---

## 2. GitHub Actions 双路线规范

### 2.1 Non-container — 基础路线

参考文件：

[Grinch27/openwrt-compiler/build-openwrt.yml](https://github.com/Grinch27/openwrt-compiler/blob/main/.github/workflows/build-openwrt.yml)（跨仓库基础路线参考）

所有新项目原则上先使用 Non-container 构建路线。

主要特征：

- 直接使用 GitHub-hosted Ubuntu Runner。
- 在 Runner 上安装所需软件包与工具链。
- 在 Runner 上完成源码获取、构建和验证。
- 使用共享 Actions 管理通用环境。
- 使用动态 Runner Image 解析机制。
- 保留必要的构建阶段、错误状态和验证证据。

Non-container 是项目最初的开发和调试路线。

只有经过实际构建验证，才能将其视为稳定基础。

不得仅凭 YAML 语法检查、单元测试或部分 Job 成功，就认定完整构建路线已经稳定。

### 2.2 Container — 进阶路线

参考文件：

[Grinch27/uboot-mtk-compiler/uboot_allinone.yml](https://github.com/Grinch27/uboot-mtk-compiler/blob/main/.github/workflows/uboot_allinone.yml)（跨仓库 Container 路线参考）

重点参考该文件的前 200 行。

重点学习：

1. Workflow 顶层配置。
2. `run-name`。
3. `permissions`。
4. `defaults`。
5. 顶层 `env`。
6. `runner-image` Job。
7. `toolchain` Job。
8. `needs` 与 `outputs`。
9. `container`。
10. YAML Anchors 与 Aliases。
11. Job 环境变量与共享 Action 的组织方式。

Container 路线的主要目的是：

- 使用更新的 Ubuntu 容器环境。
- 探索更新的 GCC、Clang、JDK 等工具链。
- 隔离宿主 Runner 与构建环境。
- 减少宿主 Runner 预装软件带来的影响。
- 在保证构建正确性的基础上研究维护优化。

优先参考 GitHub Actions 原生 Job Container：

```yaml
jobs:
  build:
    needs: runner-image
    runs-on: ${{ needs.runner-image.outputs.RUNNER_IMAGE }}
    container:
      image: ubuntu:devel
```

`ubuntu:devel` 只是追新路线的参考示例，不是所有项目必须采用的镜像。

容器发行版和工具链版本必须结合实际项目兼容性决定。

使用 `docker run` 手动运行构建容器时，需要与 Job 级 `container:` 区分。

不得无条件复制参考项目中的 `--privileged`、Docker Socket 挂载、宿主设备挂载或安全隔离关闭参数。

### 2.3 路线开发顺序

标准开发顺序：

```text
Non-container
    |
    v
完成基础构建
    |
    v
实际验证
    |
    v
形成稳定基线
    |
    v
Container
    |
    v
探索新环境与新工具链
    |
    v
构建与兼容性验证
```

Container 路线出现失败时，必须区分：

- 源码自身问题。
- 构建脚本问题。
- Runner 环境变化。
- 容器环境变化。
- 工具链版本变化。
- 工作流配置问题。

已经采用 Container 的现有项目不因本规范而自动重构。

是否补建 Non-container 路线、是否迁移为原生 Job Container，必须作为单独任务研究和批准。

---

## 3. GitHub Actions uses 引用规范

### 3.1 强制使用 main/master

所有远程 GitHub Action 的 `steps[*].uses` 必须使用 `main` 或 `master` 分支。

禁止：

- 固定 Commit SHA。
- 固定版本 Tag。
- `@v1`、`@v2`、`@v3`、`@v4`、`@v5` 等版本引用。
- 未经探测的分支引用。

正确示例：

```yaml
- name: Checkout
  uses: actions/checkout@main

- name: Upload Artifact
  uses: actions/upload-artifact@main

- name: Resolve runner-image
  uses: Grinch27/github-actions/.github/actions/resolve-runner-image@main
```

### 3.2 一次性分支探测

每次引入一个新的远程 Action 时：

1. 检查仓库是否存在。
2. 检查其默认分支。
3. 探测 `main` 和 `master` 是否存在。
4. 检查实际 Action 入口文件。
5. 检查 Action 输入、输出和运行条件。
6. 选择有效的默认分支引用。

选择规则：

- 默认分支为 `main`：使用 `@main`。
- 默认分支为 `master`：使用 `@master`。
- 两者都存在：以仓库默认分支为准。
- 均不可用：停止自动生成并报告。
- 探测失败：不得猜测。

分支探测在编写时执行一次。

正常工作流运行时，不必重复查询默认分支。

### 3.3 引用边界

本规则适用于远程 GitHub Action 的 `steps[*].uses`。

以下形式分别处理：

- 本地 Action：`uses: ./...`，不需要远程分支引用。
- Reusable Workflow：`jobs.<job_id>.uses`，需要额外核对调用语法和运行限制。
- `docker://` Action：不具备 `@main/@master` 形式，暂不自动套用分支规则。

遇到不能直接使用 `main/master` 的依赖，不得静默改用固定版本。

必须报告实际限制和可行替代方案。

### 3.4 动态依赖风险

`main/master` 是动态引用，会随上游分支更新。

因此：

- 必须核对 Action 来源。
- 不得未经批准扩大 Token 或 Secrets 权限。
- 上游接口或行为变化后，应重新执行相关验证。
- 不得将动态引用理解为依赖内容不可变。
- 如果动态依赖带来不可接受的安全或兼容性风险，应报告冲突并停止相关变更。

本规则是用户明确选择的追新策略。

---

## 4. Runner Image 统一解析规范

### 4.1 必须使用共享 Action

所有适用的 GitHub Actions 工作流必须配置：

`Grinch27/github-actions/.github/actions/resolve-runner-image@main`

标准实现：

```yaml
jobs:
  runner-image:
    name: ubuntu-latest - runner-image
    runs-on: ubuntu-latest
    outputs:
      RUNNER_IMAGE: ${{ steps.resolve-runner-image.outputs.RUNNER_IMAGE }}
    steps:
      - name: Resolve runner-image
        id: resolve-runner-image
        uses: Grinch27/github-actions/.github/actions/resolve-runner-image@main
```

### 4.2 其他 Job 必须依赖 runner-image

标准实现：

```yaml
jobs:
  build:
    needs: runner-image
    runs-on: ${{ needs.runner-image.outputs.RUNNER_IMAGE }}
```

存在多个依赖时：

```yaml
jobs:
  build:
    needs:
      - runner-image
      - toolchain
    runs-on: ${{ needs.runner-image.outputs.RUNNER_IMAGE }}
```

要求：

- `runner-image` 必须输出 `RUNNER_IMAGE`。
- 普通 Ubuntu Runner Job 必须包含 `needs: runner-image`。
- 后续 Job 必须从 `needs.runner-image.outputs.RUNNER_IMAGE` 获取 Runner。
- 不允许在后续普通 Job 中重复硬编码 Ubuntu Runner 标签。

### 4.3 引导 Job 例外

`runner-image` Job 自身允许使用：

```yaml
runs-on: ubuntu-latest
```

因为解析 Action 必须先在一个 Runner 上运行。

不得让 `runner-image` Job 依赖自己的输出。

### 4.4 特殊工作流

Reusable Workflow 的调用 Job 不直接支持 `runs-on`。

对于不支持普通 `runs-on` 结构的 Job，需要依据官方语法单独处理。

不得为了满足形式统一而生成无效 YAML。

### 4.5 Runner 与 Container 区分

- `runs-on`：GitHub Actions Job 的宿主 Runner。
- `container.image`：运行该 Job 的容器镜像。

动态 Runner 解析不意味着 Container 镜像必须使用相同版本的 Ubuntu。

---

## 5. GitHub Actions 工作流整体风格

### 5.1 顶层结构

优先采用如下组织顺序：

```yaml
name:

# GitHub Actions workflow syntax reference:
# https://docs.github.com/zh/actions/reference/workflows-and-actions/workflow-syntax
# https://docs.github.com/zh/actions/reference/workflows-and-actions/metadata-syntax#runs-for-composite-actions
# https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#yaml-anchors-and-aliases

on:

run-name:

permissions:

defaults:

env:

jobs:
```

可以根据实际需要添加：

- `concurrency`。
- Job 输出。
- Container 配置。
- Matrix。
- 其他 GitHub Actions 官方支持的字段。

不为了形式完整而强行添加无用途的配置。

### 5.2 环境变量

优先在顶层 `env` 中存放多个 Job 共用的环境变量。

参考：

```yaml
defaults:
  run:
    shell: bash

env:
  DEBIAN_FRONTEND: noninteractive
  DEBCONF_NONINTERACTIVE_SEEN: "true"
  APT_LISTCHANGES_FRONTEND: none
  NEEDRESTART_MODE: a
  FORCE_UNSAFE_CONFIGURE: "1"
  TZ: Etc/UTC
  LANG: C.UTF-8
  LC_ALL: C.UTF-8
  GIT_TERMINAL_PROMPT: "0"
```

该配置仅为参考集合。

各项目应依据实际需求决定保留哪些变量。

不得将 `FORCE_UNSAFE_CONFIGURE` 等可能影响构建行为的配置作为无条件默认值。

### 5.3 YAML Anchors 与 Aliases

对于多个 Job 重复使用的 Container 配置，优先评估 YAML Anchors。

参考：

```yaml
jobs:
  toolchain:
    needs: runner-image
    runs-on: ${{ needs.runner-image.outputs.RUNNER_IMAGE }}
    container:
      image: &container_image ubuntu:devel

  build:
    needs:
      - runner-image
      - toolchain
    runs-on: ${{ needs.runner-image.outputs.RUNNER_IMAGE }}
    container:
      image: *container_image
```

对于适合复用的 `options`、`volumes` 等，也可以采用同样方式。

YAML Anchors 应用于消除真正重复的配置，不得损害可读性或兼容性。

### 5.4 共享 Actions

优先检查：

`Grinch27/github-actions`

重点复用实际已经存在并经过核验的共享 Actions。

包括但不限于：

- `resolve-runner-image`
- `resolve-toolchain-version`
- `run-sudo`
- `env-toolchain`
- `env-indocker`

不得仅根据名称假定 Action 存在或具有某种功能。

调用之前必须阅读实际的 `action.yml` 或 `action.yaml`，检查输入、输出和运行要求。

### 5.5 权限与安全

`permissions` 根据当前项目既定授权确定。

不得仅因为参考工作流使用 `write-all`，便擅自扩大其他项目的工作流权限。

现有已授权权限不因本次规范制定而自动改变。

---

## 6. 技术文档

GitHub Actions 工作流语法：

https://docs.github.com/zh/actions/reference/workflows-and-actions/workflow-syntax

GitHub Actions Composite Action 元数据语法：

https://docs.github.com/zh/actions/reference/workflows-and-actions/metadata-syntax#runs-for-composite-actions

GitHub Actions YAML Anchors 与 Aliases：

https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#yaml-anchors-and-aliases

要求：

- 修改工作流时，优先检查官方文档。
- 不推测 GitHub Actions 不支持的语法。
- 不把参考工作流的历史写法当成官方强制规范。
- 对语法、表达式作用域和 Job 依赖进行静态验证。
- 静态检查不能替代真实工作流运行。

---

## 7. 实施与授权边界

本规范仅定义工作流设计与执行边界；是否实际修改或部署由具体任务授权决定。

未经当前任务明确授权，不自动执行：

- 工作流重构。
- Non-container 路线补建。
- Container 路线迁移。
- 依赖升级。
- Action 引用替换。
- Runner 配置修改。
- 源码修改。
- 超出当前任务范围的 Git Commit 或 Push。
- GitHub Actions 工作流触发。

新的规则用于指导后续明确授权的修改任务。

不能因为当前仓库存在与新规范不一致的代码，就自动进行迁移或删除。

<!-- END: NEW WORKFLOW RULES -->

---

# Part II — U-Boot MTK Compiler 项目专属约束

<!-- BEGIN: PROJECT-SPECIFIC RULES -->

## 1. 项目和现有工作流

本仓库使用 GitHub Actions 为 MediaTek MT7981/MT7986 设备构建 U-Boot，主要上游为 `hanwckf/bl-mt798x`。开始修改前先读取 `README.md` 和实际相关工作流。

目前 `.github/workflows/` 包含：

- `uboot_allinone.yml`：正式 U-Boot Docker 工作流；`README.md` 将其描述为唯一维护的正式入口。
- `uboot_gpt.yml`：独立的 97 变体 Matrix 试验工作流。不得因 README 的“唯一维护”表述而忽略或删除该文件。

两份文件目前均使用 Job Container，不代表已经存在或验证完成本项目的 Non-container 基础实现。Part I 为未来工作流设计标准，不能作为未经授权迁移当前正式入口的依据。

## 2. 设备配置与构建正确性

修改前检查上游 Git 分支和实际 Commit，核对设备清单、SoC、`BOARD`、`MULTI_LAYOUT`、`FIXED_MTDPARTS` 的组合语义。不得悄悄删除 Board/变体、改变多布局或固定分区参数，或将不同硬件的镜像混为同一产物。

正式工作流当前使用 `ubuntu:26.04` Container 与动态工具链；这是已读取配置，不代表全部变体已在该镜像下完成本次验证。调整 Ubuntu、GCC 或 OpenSSL 兼容性时，必须区分编译工具链错误、上游源码变化和配置差异，不以跳过失败、沿用历史镜像或临时移除变体获得绿色状态。

`README.md` 记录的 U-Boot / BL2 / FIP 与刷写信息只作相关设备说明；固件编译完成不代表目标设备可启动，更不自动授权 SSH、MTD 写入或设备刷写。

## 3. 触发与交付边界

正式工作流的当前行为以源码核验为准：schedule 全量构建具有 Release 发布路径；工作流变更 push 的回归测试、手动输入 `publish_release` 和 `repository_dispatch` 的发布条件，均需保留并逐项验证。不要把文档提交视为授权修改发布策略或启动全量构建。

试验工作流以 97 变体独立矩阵与 Artifact 验证为主；不得把单变体验证、Artifact 上传或部分 Job 成功称为所有变体成功。变更构建矩阵、共享源码快照或并行限制时，需检查完整组合数、失败范围和产物身份。

新增发布、覆盖 Release、修改权限、引入付费 Runner、修改生产构建入口或设备操作，需要相应当前授权；本次仅授权新建 `AGENTS.md` 并提交 `main`。

## 4. 验收与执行

先执行 YAML/actionlint、Shell 语法、Action 分支和 Runner 依赖检查；涉及构建逻辑时在授权范围内执行真实 CI，并验证编译退出码、每个预期目标的输出、尺寸及 SHA256。历史 97 变体成功记录不能无条件迁移到后续 SHA。

所有结论必须明确关联 Commit、Run、设备/变体和实际产物；未做实机启动验证时标注 `NOT_TESTED`。不得重写 Git 历史、删除试验记录或扩大权限以满足表面一致性。

<!-- END: UBOOT-MTK-COMPILER PROJECT RULES -->
