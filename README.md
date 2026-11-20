# DFAC · DeepSeek Full-stack AI Client

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Version](https://img.shields.io/badge/version-1.2.0-brightgreen)
![Platform](https://img.shields.io/badge/Android-26%2B-3DDC84)

> 全名：**基于 DeepSeek 全家桶和 LLM 的全栈 AI 客户端**
> 缩写取英文名 `DeepSeek Full-stack AI Client` 的首字母 → **DFAC**（读作 "D-F-A-C"）。

按架构图 1:1 落地的 **Android 原生聊天 App**：Room 存消息 → token 估算 → WorkManager 异步摘要 → summary_cache 冻结 → 组装请求 → OkHttp SSE 流式回复 → usage / prompt cache 命中率自检 → 状态渲染。

技术栈刻意贴合原图：**Kotlin + Room + WorkManager + OkHttp + SSE**，没有引入 Compose / Hilt / Retrofit 等会改变链路形态的东西。

> 包名仍为 `com.memorychat`（改包名会让所有 import 和安装身份一起变，不影响使用；如需彻底换成 `com.dfac` 可以再做一次全量重构）。

---

## 1. 交付物

| 产物 | 位置 |
| --- | --- |
| 可安装 APK | 运行 `scripts/build-apk.ps1` 生成 `dist/DFAC-1.2.0-debug.apk`，或从 GitHub **Releases** 下载 |
| 完整源码 | [`app/`](app) |
| 架构图 ↔ 代码节点映射 | [`docs/architecture-mapping.md`](docs/architecture-mapping.md) |
| 单元 / Robolectric / MockWebServer 测试（42 个用例） | [`app/src/test/`](app/src/test) |
| 版本记录 | [`CHANGELOG.md`](CHANGELOG.md) |
| 构建 / 安装脚本 | [`scripts/build-apk.ps1`](scripts/build-apk.ps1)、[`scripts/install-apk.ps1`](scripts/install-apk.ps1) |

APK 参数：`applicationId=com.memorychat.debug`、`versionName=1.2.0-debug`（versionCode 3）、`minSdk=26`、`targetSdk=35`、debug 签名（可直接安装）。

> APK 属于构建产物，已加入 `.gitignore`，请通过 **GitHub Releases** 分发，不要提交进 git 历史。

---

## 2. 快速开始（真机）

1. **安装**：从 Releases 下载 APK，或本地构建（需 JDK 17 + Android SDK 35）：
   ```powershell
   ./gradlew :app:assembleDebug
   # 或一键脚本（会同时拷贝到 dist/）
   powershell -ExecutionPolicy Bypass -File scripts/build-apk.ps1
   adb install -r dist/DFAC-1.2.0-debug.apk
   ```
   > 同一个 applicationId，直接覆盖安装即可；本地会话、已填的 Key、已缓存的模型列表都会保留（数据库有 v1→v2 迁移，设置有多版本迁移）。
2. **聊天**：打开 App 直接发消息。**不填 Key 也能跑**，会自动进入离线 Mock 模式，完整演示压缩、重试、缓存体检、状态条。
3. **打开侧边栏**：从屏幕**右缘向左滑**，或点右上角滑块图标。侧边栏从右侧弹出，顶部两个页签：
   - **历史对话**（默认）：列出所有会话，点按切换、长按删除、「新建对话」一键开新会话
   - **设置**：API Key、模型下拉、阈值、temperature、max_tokens、思考/工具开关等
4. **接真实 DeepSeek**：设置页填 `sk-...` 的 API Key → 模型用**下拉选择** →「保存」→「测试连接」。
   「测试连接」会调官方 `GET /models`，把**你的 Key 真实可用的模型列表持久化**，之后下拉框直接展示这份列表；发现当前模型不可用会自动切换。
5. **验证压缩链路**：设置页把「压缩阈值」调小（比如 300）、保留轮数保持 3，聊几轮就能看到顶部 chips 出现
   `压缩 X 轮 · 省 Y` 和缓存命中率；命中率偏低时会出现黄色体检告警条。

> **LLM 总开关**在侧边栏设置页顶部：关掉后消息照常保存，但**不会发起任何 DeepSeek 请求**、也不会触发摘要任务，适合省额度/断网调试。开关立即生效。
>
> **联网搜索 / 深度思考**两个快捷开关在输入框上方（对话下方）：联网搜索对应官方服务端工具 `deepseek_web_search`（仅 `deepseek-flash`，开启时自动切模型），深度思考对应 `thinking`；两个都开就是官方的「深度思考搜索」。搜索过程会显示「联网搜索：关键词 / 获得 N 条来源」，回复后状态条会列出引用来源标题。
>
> **字数太少**的根因不是上下文窗口，而是旧版 `max_tokens=2048` + 人设限制。现在默认 `max_tokens=4096`、人设允许展开长回复，设置页可调到 8192 甚至更高（官方上限 393216）。

---

## 3. 架构图 ↔ 代码

完整逐节点映射见 [`docs/architecture-mapping.md`](docs/architecture-mapping.md)，主干：

```
用户输入消息 ─────────────────────────────── ui/chat/ChatActivity + ChatViewModel
  → 写入 Room messages ────────────────────── data/local/{Entities,Daos,AppDatabase}
  → 估算历史 token ────────────────────────── domain/TokenEstimator
  → > 阈值 ? 触发压缩 ─────────────────────── data/repo/ChatRepository#completeTurn
      → 计算历史前缀 hash ─────────────────── domain/PrefixHash
      → summary_cache 命中? ───────────────── data/local/SummaryCacheDao
      → 未命中 → WorkManager 异步生成 ─────── work/SummaryWorker
      → 完成 → 写入并冻结摘要 ─────────────── work/SummaryWorker + SummaryCacheEntity.frozenAt
      → 失败/超时 → 原文转发 fail-open ─────── data/repo/ChatRepository（degraded）
  → 组装请求 system + 摘要 + 最近 N 轮原文 ── domain/PromptAssembler
  → 摘要指令放最后一条 user 消息 ──────────── PromptAssembler#attachDirective
  → 注入温度 / 人设 / thinking / 工具参数 ─── data/settings/AppSettings + domain/ToolCatalog
  → OkHttp SSE ────────────────────────────── network/DeepSeekHttpClient
      → 200 逐 chunk 更新气泡 ─────────────── ChatRepository + ChatViewModel#onAssistantText
      → 429/5xx 指数退避重试 ≤3 ───────────── domain/RetryPolicy + DeepSeekHttpClient
      → 4xx 不可重试 → 错误态可重发 ───────── ChatRepository#failTurn + 气泡点击重发
      → 联网搜索事件 / 引用来源 ───────────── server_tool_event + annotations/sources
  → 流结束写完整回复 + 读 usage ───────────── network/DeepSeekHttpClient#parseUsage
  → prompt_cache_hit_tokens 命中率正常? ───── domain/CacheDiagnostics（PromptCacheInspector）
      → 偏低 → 标记会话 + 扫动态内容 ──────── DynamicContentDetector
  → 更新压缩状态条 ────────────────────────── SessionDao#bumpCompression + ChatViewModel
  → UI 渲染完成 ───────────────────────────── MessageAdapter
```

---

## 4. DeepSeek 官方接口对齐（2026 现行文档）

> 依据 <https://api-docs.deepseek.com/zh-cn/> 核对，早期版本里的 `deepseek-chat` / `deepseek-reasoner` **已下线**，
> 这就是之前"模型调用不到"的根因。

| 项 | 官方现行值 | 本工程实现 |
| --- | --- | --- |
| 模型 | `deepseek-flash`（默认）、`deepseek-v4-pro` | 下拉选项 + `GET /models` 运行时校正并持久化；设置迁移把旧模型名纠正为 `deepseek-flash` |
| Base URL | `https://api.deepseek.com` | 默认值；`apiRoot()` 兼容带 `/chat/completions`、`/models` 的写法 |
| 对话接口 | `POST /chat/completions` | `DeepSeekHttpClient#buildPayload` |
| 模型列表 | `GET /models` | `DeepSeekHttpClient#listModels`；「测试连接」校验并自动切模型，结果持久化 |
| 思考模式 | `thinking: {type: enabled/disabled}`，官方默认开启 | 侧边栏开关 + 输入框快捷开关；关闭时显式下发 `disabled`（响应快、temperature 生效） |
| 思考强度 | `reasoning_effort: low/high/max` | 下拉选择，仅在思考模式下发 |
| 推理内容 | `delta.reasoning_content` | 单独解析为 `StreamEvent.Reasoning`，只在状态条显示进度，不污染气泡 |
| 联网搜索 | `tools: [{"type": "deepseek_web_search"}]`，服务端执行，仅 `deepseek-flash` | 输入框上方「联网搜索」开关；与 function 工具互斥、不设 `tool_choice`；解析 `server_tool_event` 与 `annotations/sources` |
| max_tokens | 1 ~ 393216；非思考默认 8K，思考默认 64K | 默认 4096；思考模式自动抬到 ≥8192，界面上限 393216 |
| 流式 usage | `stream_options.include_usage`，usage 挂在最后一个内容块 | 逐块解析，取 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` |
| 缓存字段 | `prompt_cache_hit_tokens`、`prompt_cache_miss_tokens`、`prompt_tokens_details.cached_tokens` | 三者都兼容 |
| 错误码 | 400/401/402/408/413/422/429/5xx | `humanReadableError()` 逐类中文提示；429/5xx 指数退避，其余不重试 |

---

## 5. 关键实现决策

| 决策 | 原因 |
| --- | --- |
| 前缀 hash 只吃 `role + 内容`（归一化换行/行尾空白），**剔除 id、时间戳** | 否则每轮 hash 都变，`summary_cache` 永不命中，压缩就白做。`PrefixHashTest` 锁死该约束 |
| 架构图的 `system + 摘要` 合并成**一条** system 消息 | 「静态前缀」字节完全稳定，对 prompt cache 更友好 |
| 摘要指令**追加到最后一条 user 消息尾部** | 与图一致，且只改最后一条消息尾部，前面的历史前缀不变 |
| 流式**已输出内容后不再重试**，只有"零输出"的 429/5xx 才退避 | 否则重试会把重复文本拼进同一个气泡 |
| WorkManager 等待上限 = `摘要超时 + 3s`，超时 fail-open 用原文 | 与图一致；**后台任务不取消**，跑完仍写 `summary_cache`，下一轮命中 |
| 流式期间每 350ms 才落一次 Room | 每 chunk 写库会卡；内存 sink 保证气泡实时，DB 保证进程被杀后有内容 |
| usage 通过 `stream_options.include_usage` 显式索取 | DeepSeek 流式默认不回 usage，拿不到 `prompt_cache_hit_tokens` 就无法做缓存体检 |
| LLM 总开关在「估算 token」之后直接短路 | 关闭时不组装请求、不入队摘要 Worker，只落库 + 本地提示 |
| 联网搜索与 function 工具互斥、不设 `tool_choice` | 官方明确要求；同时避免 400 |
| 会话切换 = 换 Room 查询源 + 换统计 | 历史对话互不串台；删除会话连带清消息与摘要缓存 |

---

## 6. 工程结构

```
memory-chat/
├── app/src/main/java/com/memorychat/
│   ├── DfacApp.kt                  Application 入口（DFAC）
│   ├── di/ServiceLocator.kt        轻量依赖容器（如要换 Hilt 只需改这里）
│   ├── domain/                     纯 Kotlin，可跑 JVM 单测
│   │   ├── TokenEstimator.kt       token 估算
│   │   ├── PrefixHash.kt           历史前缀 hash（SHA-256）
│   │   ├── PromptAssembler.kt      切分 / 组装 / 摘要指令注入
│   │   ├── CacheDiagnostics.kt     usage 命中率 + 动态内容探测
│   │   ├── RetryPolicy.kt          指数退避
│   │   ├── ToolCatalog.kt          全家桶工具 schema
│   │   └── LocalToolRuntime.kt     本地工具执行
│   ├── data/
│   │   ├── local/                  Room：messages / sessions / summary_cache
│   │   ├── settings/               AppSettings + SharedPreferences（含模型列表缓存）
│   │   └── repo/ChatRepository.kt  全链路编排（核心文件）
│   ├── network/                    OkHttp SSE + Mock + 动态切换
│   ├── work/                       SummaryWorker + SummaryScheduler
│   └── ui/chat                     聊天页 + 右侧侧边栏（历史对话 / 设置 两页签）
├── app/src/test/java/...           42 个测试（domain + Robolectric + MockWebServer）
├── docs/architecture-mapping.md    逐节点映射
└── scripts/                        构建 / adb 安装脚本
```

### UI 交互一览

| 区域 | 内容 |
| --- | --- |
| 顶部 | 状态文案（估算/压缩/思考/联网/重试/降级）+ 4 个 token 消耗 chip（上下文 / 本轮 / 缓存 / 压缩节省） |
| 中部 | 气泡列表；助手气泡下方显示「输入 / 输出 / 缓存」；错误气泡可点重发 |
| 输入区上方 | 「联网搜索」「深度思考」两个快捷开关（与设置页开关双向同步） |
| 侧边栏（右侧弹出） | 历史对话页：会话列表、切换、长按删除、新建；设置页：Key、模型下拉、阈值、温度、max_tokens、思考开关、工具开关、测试连接 |
| 键盘 | 用 `WindowInsets.ime()` 给根布局加底部内边距，输入框不会被键盘挡住 |

---

## 7. 构建与测试

要求：**JDK 17**、Android SDK（`platforms;android-35`、`build-tools;35.0.0`）、Gradle 由仓库内 wrapper 提供（8.11.1）。

```powershell
# Android Studio：直接 Open 本目录，Sync 后 Run
# 命令行：
cd memory-chat
./gradlew :app:assembleDebug          # 产物 app/build/outputs/apk/debug/app-debug.apk
./gradlew :app:testDebugUnitTest      # 42 个测试
```

`local.properties` 里的 `sdk.dir` 是本次构建环境的路径，**Android Studio 打开时会自动改写**；命令行构建时按自己机器改一下即可（该文件已被 `.gitignore` 忽略）。

本次验证结果（Windows + JDK 17.0.21 + AGP 8.7.3 + Gradle 8.11.1 + SDK 35）：

- `:app:assembleDebug` ✅ APK 已产出（`dist/DFAC-1.2.0-debug.apk`，`apksigner verify` 通过，`aapt2 dump badging` 显示 `versionCode=3` / `versionName=1.2.0-debug` / 应用名 DFAC）
- `:app:testDebugUnitTest` ✅ **42/42 通过**
  - 27 个 domain 纯 JVM 单测：token 估算、hash 稳定性（id/时间戳/空白归一化）、切分与组装、摘要指令注入、缓存命中诊断与动态内容探测、退避策略
  - 6 个 Robolectric 运行时冒烟测试（不需要真机/模拟器）：
    1. 离线 Mock 模式下一轮对话完整跑通：Room 建库 → SSE 流式回调 → 气泡落库 complete → usage 写回 → 缓存命中率记录
    2. 超阈值时命中已冻结摘要：走压缩分支、累计「已压缩轮数 / 节省 token」
    3. `SummaryWorker` 真实执行：生成摘要并冻结写入 `summary_cache`
    4. **LLM 总开关关闭**：不发请求、不留错误，只落库 + 本地提示
    5. 旧版设置迁移：偏小参数抬高 + 已下线模型名纠正为 `deepseek-flash`（合法的 `deepseek-v4-pro` 保持不变）
    6. **历史对话**：会话列表、消息互不串台、删除会话连带清空消息
  - 9 个 MockWebServer 客户端测试：`complete()` 必须在 IO 线程执行（回归 `NetworkOnMainThreadException`）、非 200 抛 `ApiException`、SSE 增量与 usage 解析、`reasoning_content` 与 `content` 分流、请求体带 `model` / `thinking` / `reasoning_effort`、`GET /models` 解析、联网搜索声明 `deepseek_web_search` 且不带 `tool_choice`、`server_tool_event` 与引用来源解析

---

## 8. 已知限制

1. **真实 Key 的端到端请求没在真机上跑过**（构建机无 adb 设备）：链路本身已由 Mock + Robolectric + MockWebServer 覆盖；配好 Key 后点设置页「测试连接」即可当场验证真实接口与可用模型。
2. 思考模式：官方默认**开启**思考，本 App 聊天默认**关闭**（快、temperature 生效），侧边栏可开启并选 `reasoning_effort`；推理内容不写进气泡，只显示进度。
3. API Key 用私有 `SharedPreferences` 保存，未接 `EncryptedSharedPreferences` / Keystore；如需更高安全等级可替换 `SettingsRepository`。
4. token 估算是启发式（CJK 0.62 token/字、拉丁 3.6 字符/token + 每条 4 开销），只用于压缩决策，不追求与分词器完全一致。
5. 工具调用只做 1 轮回灌，没有多轮 agent 循环；工具本身只读（查时间 / 检索记忆 / 查能力）。
6. 摘要 Worker 用「历史前缀 hash 校验 + 唯一任务名 KEEP」防重复；如果两次压缩间隔极短且历史正在变化，本轮可能 fail-open 降级，下一轮会命中缓存——这是设计内行为。
7. 联网搜索的多轮上下文：官方建议把上一轮 assistant 消息连同 `server_tool_calls` 原样回传（`signature` 用于把读过网页拼回上下文）。本工程目前只回传正文，所以**多轮里之前搜到的网页不会重新进入上下文**（请求仍正常，模型会按需重新搜索）。
8. 联网搜索仅支持 `deepseek-flash`（官方要求），开启时自动切模型；`deepseek_web_search` 与 function 工具互斥，同时开启时只发服务端搜索工具。
9. 历史对话列表显示标题（首条消息）、更新时间与压缩统计，暂未做「最后一条消息预览」。

---

## 9. 版本记录

完整变更见 [`CHANGELOG.md`](CHANGELOG.md)。

| versionName | versionCode | 摘要 |
| --- | --- | --- |
| 1.2.0-debug | 3 | 侧边栏「历史对话 / 设置」两页签、会话切换/删除、模型列表持久化 |
| 1.1.0-debug | 2 | 对齐 DeepSeek 2026 官方文档、联网搜索、思考模式、token 组件、键盘避让 |
| 1.0.0-debug | 1 | 按架构图实现首版压缩链路 |

---

## 10. 发布到 GitHub

仓库已按「可直接 push」整理好（`.gitignore` / `.gitattributes` / `LICENSE` / `CHANGELOG` / `CONTRIBUTING` / GitHub Actions CI），本地构建产物与密钥均不会入库。

```bash
# 1) 在 GitHub 新建空仓库（不要勾选 README / LICENSE），然后：
git remote add origin git@github.com:<your-name>/DFAC.git
git branch -M main
git push -u origin main
```

推送后 GitHub Actions（`.github/workflows/android-ci.yml`）会自动运行 42 个测试并构建 APK，产物可从 Actions 的 Artifacts 下载。

**发布 APK 到 Releases：**

```bash
./gradlew :app:assembleDebug
cp app/build/outputs/apk/debug/app-debug.apk dist/DFAC-1.2.0-debug.apk
gh release create v1.2.0 dist/DFAC-1.2.0-debug.apk \
  --title "DFAC v1.2.0" \
  --notes-file CHANGELOG.md
```

**仓库关键文件：**

| 文件 | 作用 |
| --- | --- |
| `.gitignore` | 排除 `build/`、`.gradle/`、`local.properties`、`dist/`、APK、签名文件 |
| `.gitattributes` | 统一 LF / Windows 脚本 CRLF / 二进制识别 |
| `.editorconfig` | 缩进、编码、行尾规范 |
| `LICENSE` | MIT |
| `CHANGELOG.md` | 版本记录 |
| `.github/workflows/android-ci.yml` | CI：JDK 17 + Android SDK → 测试 → 构建 APK → 上传 Artifacts |
| `CONTRIBUTING.md` | 开发与提 PR 约定 |
