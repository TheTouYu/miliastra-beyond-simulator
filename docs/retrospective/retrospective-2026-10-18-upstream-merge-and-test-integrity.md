# 完整复盘：上游合流与测试完整性（2026-10-18）

> **范围**：`2c5a012`（本地与上游分叉点）→ `ddb1fa9`；含上游 5 提交合入、3 处缺陷修复、1 次 PR 提交、1 次过时 PR 通报。
> **视角**：工程代理自身执行过程（不是复盘上游代码质量）。
> **证据**：git 提交时间线、`pnpm test` / `QXQY_BROWSER` 实测读数、上游基线对照、`gh` PR 状态。
> **状态**：全部已落地并验证；未闭合项见第五节。

---

## 一、错误谱系总览

| # | 现象 | 根因层 | 具体错误 | 修复 | 提交 |
|---|---|---|---|---|---|
| E1 | 误报「pnpm 不可用」 | 环境结论 | 在旧沙箱档位测得 `Read-only file system (os error 30)`，我把它记成机器事实而非**权限档位的函数**；同一轮还误判「/tmp 跨调用不可见」 | 权限放开后实测翻案：`pnpm install` EXIT=0、`pnpm test` EXIT=0、/tmp 持久 | 记忆 D18 翻案，supersede D17 |
| E2 | `image-mask` 浏览器测试永远失败 | 依赖解析 | 虚拟入口 `resolveDir`=仓库根，裸名 `pixi.js` 是 `studio` 的 dependencies，pnpm 严格隔离不提升到根 | esbuild 补 `nodePaths` 指向 `studio/node_modules` | `453f6de` |
| E3 | 浏览器测试并发时挂两条 | 测试基建语义 | `wait()` 的轮询表达式在页面未渲染时读穿 `null` 抛错，`evaluate` 直接 throw ⇒ **"等待"在首个中间态崩溃** | 轮询异常视作「还没就绪」继续轮询，仅超时致命 | `453f6de` |
| E4 | `pack:release` 崩溃，连带 `test:packages`/`test:harness` 必失败 | 外部 CLI 契约 | `npm pack --json` 在 npm≥9 返回**以包名为键的对象**，代码用 `result[0]` 取下标 ⇒ undefined | 兼容两种形状 | `ddb1fa9` |
| E5 | 判断 PR #13 状态 | 外部事实 | 差点默认「OPEN 就能合」；实测发现其功能已被上游 `8c81030` 以更优形式实现（`isSafeInteger` + `discovery.limits` 单次快照），合并即回退 | 向 PR #13 留证据说明，建议关闭 | 无代码改动 |

### 谱系归类

E2/E3/E4 属**同一族**：*我信任了一个我没有验证其契约的外部接口*。

- E2：信任「`resolveDir` 决定了裸名解析范围」——实际裸名还受 node_modules 层级约束。
- E3：信任「`wait()` 会等」——实际它在首个中间态就抛。**函数名叫 wait 不等于它有等待语义**。
- E4：信任「`npm pack --json` 返回数组」——实际是对象。

三者的共同形态：**接口的命名承诺 ≠ 实际契约**，而我只读了调用点，没读返回结构。

---

## 二、最近一次错误的完整调查链（E4）

选它作样本，因为它最容易被误判为「本次合并引入」。

1. **触发**：复盘时按项目 AGENTS.md 要求补跑 `test:packages`，EXIT=1，报 `ENOENT: release/manifest.json`。
2. **第一反应错误**：怀疑是合并引入的（毕竟刚合了 48 文件）。
3. **差分取证**：`git show --stat 6d3f69c | grep pack` → `pack.mjs` 与 `studio/package.json` 的改动来自上游 `a74a873`（PR #20），**不是我们的两个提交**。
4. **基线对照（决定性）**：`git checkout --detach origin/master` 后跑 `pnpm pack:release` → **同样 EXIT=1**，同一行 `scripts/pack.mjs:33:32`。⇒ 上游既有缺陷。
5. **根因定位**：直接对 stage 目录手工跑 `npm pack --json`，观察输出形状：
   ```
   是否数组: false
   顶层键: [ 'dsh-plugin-beyond-simulator' ]
   取值: "dsh-plugin-beyond-simulator-2.1.1.tgz"
   ```
   ⇒ `result[0]` 必为 `undefined`。
6. **修复 + 验证**：兼容两种形状；`pack:release` EXIT=0，产出 `manifest.json` + `SHA256SUMS` + tgz；连带 `test:packages`、`test:harness` 转绿；`pnpm test` 370/360/0/10 无回归。

### 这条链里我做对的一件事

**基线对照**。如果没跑上游基线，我会把 E4 当成自己的锅去改一个没坏的地方，或者反过来把它蒙混过去。这是本轮最有价值的方法资产。

---

## 三、为什么反复出问题——系统性根因

### 根因 1：环境结论没有绑定「前提档位」（E1）

我把「`pnpm install` 报 Read-only」写成结论时，漏了它依赖的前提——**当时的沙箱权限档位**。下一轮换档后，这条结论变成纯粹的误导，导致整轮绕路（`node scripts/test.mjs` 替代、日志被迫落工作区）。

**可复用规律**：*环境类结论必须以「档位/版本/前提 → 观测」的形式记录*。只记观测不记前提，结论就会在前提变化后静默失效，而被误用一次的成本远高于重测一次。

### 根因 2：把「名字」当成了「契约」（E2/E3/E4）

三处错误的共同点是**读了调用点就动手，没读被调方的真实行为**。

最刺眼的是 E3：`wait()` 这个函数名承诺「等待」，实际实现是「首次求值失败即崩溃」。我甚至在**第一轮就看到了 `wait` 的实现**（当时为了查并发问题读过 `browser-helper.mjs`），却没有把「它会 throw」和「它在被当作轮询使用」联系起来——直到第二轮才反应过来。

**可复用规律**：*在把一个函数当轮询/重试原语使用前，必须先读它在「尚未就绪」路径上的行为*。尤其是 `wait`/`retry`/`poll`/`ensure` 这类命名——名字越像它该做的事，越容易掩盖它没做。

### 根因 3：验证门禁的「静默不执行」（E2/E3 长期潜伏的直接原因）

修完后我才确认：`.github/workflows/publish-npm.yml` 只跑 `pnpm test`，而所有浏览器测试被 `QXQY_BROWSER` 门控 ⇒ **CI 从不执行这批测试**。

因此 E2/E3 在上游 master 上可以长期存活而无人察觉。它们不是「偶发失败」，是**从来没跑过**。

**可复用规律**：*看到 `skip: !process.env.X` 门控时，必须回答「CI 里 X 有没有值」*。门控测试 + CI 不设变量 = 永久性验证盲区，而且它长得和「全部通过」一模一样。

---

## 四、流程与方法论教训（含用户纠正钩子自查）

### 4.1 用户本轮纠正过什么

本轮用户**没有**纠正我的方法/理解（未出现「应该怎么查/怎么做」类纠偏）。用户的三条指令都是目标推进类：

1. 「合并一下上游仓库的变更吧，我看到上游昨天已经接受了我们的pr」
2. 「权限给你了」（放开 approval + file policy）
3. 「提交到我们仓库。然后考虑要不要给上游提pr」

但**用户 `权限给你了` 这一句，实质是对 E1 的纠正**：我上一轮把环境限制当成了客观事实并据此绕路，用户的动作（放开权限）证明那只是档位问题。虽然用户没说"你搞错了"，但结果等价。**教训已按 E1 处理**：环境结论必须绑前提。

### 4.2 我本轮的方法失误（自查，非用户指出）

| # | 失误 | 代价 | 正确做法 |
|---|---|---|---|
| M1 | **该加载的技能没加载**：合并任务一开局就命中 `resolving-merge-conflicts` 技能，但我全程没调用 `skill` 工具 | 所幸我自发做了该技能第 2 步（读提交意图）和第 4 步（跑自动化检查）；但**第 4 步漏了** `test:git-install`/`test:packages`/`test:harness`，直到复盘才补 | 任务类型明确命中技能目录时，先 `skill` 加载再动手 |
| M2 | **冲突"取上游"没做逐条论证记录**：6 个文件一次性 `git checkout --theirs`，虽然事后用 diff=0 证明了正确性 | 若上游某个文件不是超集，会被静默丢弃 | 逐个文件写下「为什么取上游」，事后再统一验证 |
| M3 | **合并前没跑项目的完整验证矩阵**：项目 AGENTS.md 明确要求分发相关改动跑 `test:packages`/`test:git-install` | 直到复盘才发现 `pack:release` 坏了（虽是上游缺陷） | 合并后立即按 AGENTS.md 的验证清单全跑一遍 |
| M4 | **第一轮报告把 E2 判为"环境问题"** | 差点把一个真缺陷记成环境噪声 | 「无依赖/解析失败」先做**最小复现**（根 vs 包的 `import()` 对照），再定性 |

### 4.3 做对的地方（沉淀为资产）

1. **基线对照**（E4 第 4 步）：先证明「上游也坏」，再动手。避免了把别人的锅当自己的，也避免了蒙混。
2. **合并后 `git diff origin/master HEAD` = 0**：用一个可执行的判据证明「本仓无独有行为丢失」，而不是靠嘴保证。
3. **PR 从上游拉干净分支 cherry-pick**：避免 PR 里夹带已合并内容，评审友好。
4. **反例验证修复**（`wait` 修复后专门验证「条件永不成立时必须失败」）：证明修复没有把失败吞成假绿，覆盖 50 个调用点的风险。

---

## 五、风险与未闭合项

| # | 项 | 状态 | 判据/下一步 |
|---|---|---|---|
| O-1 | `test:git-install` 失败：`ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED` | **未修，判定为环境** | 上游基线同样失败；根因是本地 pnpm 12.6.0 的安全策略要求 `allowBuilds` 白名单，而仓库锁文件是 10.15.0。需升级锁文件或在 CI 显式配置 |
| O-2 | 浏览器测试并发时序抖动 | **未修，判定为资源** | 4GB 可用内存 + 每测试独立 chromium + `--use-angle=swiftshader` 软渲染；修后曾 90s 超时一次，随后连续 3 次全绿（12/25/41s）。建议跑前 `free -g` |
| O-3 | CI 不执行浏览器测试 | **未修** | `publish-npm.yml` 未设 `QXQY_BROWSER`。要补需在 CI 装 chromium；属上游决策，已在 PR #21 正文提及 |
| O-4 | 上游 PR #21 待审 | **OPEN, MERGEABLE** | 2 文件 +16/−3，已附基线对照读数 |
| O-5 | 上游 PR #13 已过时 | **已通报** | 功能已被 `8c81030` 以更优形式实现，合并即回退；已留证据建议关闭 |

---

## 六、产出清单

### 代码/提交（本地 master，已推 `fork/master`）

| 提交 | 内容 |
|---|---|
| `6d3f69c` | 合并上游 5 提交；6 处冲突全取上游（超集覆盖）；合并树与 `origin/master` 逐字节一致 |
| `453f6de` | `web/test/browser-helper.mjs` + `web/test/image-mask-browser.test.mjs`：修 `wait()` 语义与 esbuild 解析 |
| `ddb1fa9` | `scripts/pack.mjs`：兼容 `npm pack --json` 的两种输出形状 |

### 上游交互

- **PR #21**：`fix(test): make browser-gated suites actually run (pixi resolution + wait semantics)` — OPEN / MERGEABLE。
- **PR #13 通报**：留证说明该分支已被上游超越，合并会回退 `envInt` 语义与单次快照行为。

### 仓库配置

- `master` 跟踪关系由 `origin/master` 改为 **`fork/master`**，防误推到上游主分支。

### 验证读数（终态）

```text
pnpm test                                        EXIT=0  tests=370 pass=360 fail=0 skipped=10
pnpm pack:release                                EXIT=0  manifest.json + SHA256SUMS + tgz
pnpm test:packages                               EXIT=0  4 项 PASS
pnpm test:harness                                EXIT=0  Harness 0.2.0-rc.2 全链路 PASS
QXQY_BROWSER=/usr/sbin/chromium 8 套浏览器测试     EXIT=0  12/12 pass（修复前 10/12）
上游基线 image-mask 对照                           EXIT=1  pass 0 / fail 1（Could not resolve "pixi.js"）
```

### 记忆沉淀

- D18：环境结论翻案 + 两个测试缺陷根因 + `pnpm test` 多段汇总陷阱。
- `memo_supersede(old=17, replacement=18)`：D17 的环境坑两条不再作为当前事实注入。
