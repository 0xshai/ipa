# 单词音标查询与高亮功能设计方案

## Summary

为现有单页音标学习网站新增**单词音标查询功能**：在顶部添加搜索输入框，用户输入任意英文单词后，调用免费在线词典 API 获取该单词的 UK/US 音标，将音标字符串解析为单个音标符号，在结果面板中按读音分组（短元音/长元音/双元音/清辅音/浊辅音/鼻音近音）以不同颜色展示；同时自动高亮下方音标表格中对应的音标卡片，方便用户快速定位并进行发音练习。

## Current State Analysis

现有网站为纯前端单页应用（`index.html`），核心特征：

- **UI 风格**：暗色科技风，使用 CSS 变量（`--bg`, `--cyan`, `--green` 等），有网格背景、光晕 hover 效果
- **布局结构**：顶部 `topbar`（标题 + GB/US 口音切换按钮）→ 下方 `#chart`（元音、辅音两个分类表格）
- **数据模型**：`categories` 数组硬编码 44 个音标，每个音标项含 `s`(符号), `w`(示例词), `p`(示例词音标), `f`(单词音频文件名), `ipa`(音标音频文件名)
- **交互**：点击音标卡片上半部播放音标发音，点击下半部（单词/音标文本）播放单词发音（UK/US 根据 `state.accent` 切换）
- **技术栈**：纯 HTML + CSS + 原生 JS，无构建工具，无外部依赖

**缺失能力**：
- 无法查询任意单词的音标
- 音标项无唯一标识，难以从音标字符串反向定位卡片
- 无音标符号解析/拆分逻辑
- 无按读音特征分组的颜色体系

## Proposed Changes

所有改动集中在 `index.html` 的 `<style>` 和 `<script>` 区域内，保持项目零依赖、单文件结构。

### 1. 数据层扩展

**文件**：`index.html` `<script>` 顶部

**内容**：
- 为 `categories` 中每个 `item` 增加 `id` 字段，值为音标符号去除 `/` 后的字符串（如 `"ɪ"`, `"eɪ"`, `"tʃ"`）
- 新增 `IPA_GROUPS` 常量对象，定义 6 个读音分组及其包含的音标 `id`、颜色、标签：
  - 短元音（`#39d5ff` cyan）
  - 长元音（`#628bff` blue）
  - 双元音（`#35f0ac` green）
  - 清辅音（`#ff6b6b` red）
  - 浊辅音（`#ffd166` yellow）
  - 鼻音/近音（`#a78bfa` purple）
- 新增 `ALL_IPA_IDS` 数组：所有 44 个音标 `id` 按字符串长度降序排列（确保多字符音标如 `eɪ`, `tʃ` 优先匹配）
- 新增 `US_TO_UK_MAP` 对象：`{ 'oʊ': 'əʊ', 'ɚ': 'ə', 'ɝ': 'ɜː', 'ɑ': 'ɒ' }`，用于将 API 返回的美式音标符号映射到本站数据标准
- 在 `render()` 渲染完成后，构建 `ipaRegistry` Map：`id` → `{item, element, category}`，便于后续高亮时快速查找 DOM 元素

### 2. 音标解析引擎

**文件**：`index.html` `<script>` 中（数据层之后）

**内容**：
- `parseIpaString(ipaStr, accent)` 函数：
  1. 清洗：去除首尾 `/`，去除重音符号 `ˈ` `ˌ`
  2. 标准化：若 `accent === 'us'`，按 `US_TO_UK_MAP` 替换符号（多字符映射优先）
  3. 贪婪匹配：遍历清洗后的字符串，用 `ALL_IPA_IDS` 从长到短尝试匹配，匹配成功则记录 `id` 并前进对应长度，未识别字符则跳过
  4. 返回 `tokens` 数组（如 `['h','ə','l','əʊ']`）
- `getGroupById(id)` 辅助函数：遍历 `IPA_GROUPS` 返回对应分组对象，找不到返回 `null`

### 3. API 调用层

**文件**：`index.html` `<script>` 中

**内容**：
- `fetchWordPhonetics(word)` 函数：
  - 请求 `https://api.dictionaryapi.dev/api/v2/entries/en/${encodeURIComponent(word.toLowerCase())}`
  - 使用 `AbortController` 设置 8 秒超时
  - 非 200 状态码抛出对应错误（404 → "未找到该单词"，其他 → "查询失败，请稍后重试"）
- `extractPhonetic(data, accent)` 函数：
  - 从 API 返回数组取第一个 `entry`
  - 在 `entry.phonetics` 中查找含 `text` 的条目
  - 优先按口音特征匹配（US 找含 `oʊ`/`ɑ` 的，UK 找含 `əʊ`/`ɒ` 的）
  - 无匹配则 fallback 到第一个有 `text` 的条目
  - 返回 `{ word, phoneticText, audioUrl }`

### 4. 搜索 UI 组件

**文件**：`index.html` `<style>` + `<body>` 顶部结构

**内容**：
- 修改 `.topbar-inner` 布局为响应式三列：`1fr 320px 200px`（标题区 | 搜索框 | 口音切换）
- 移动端（`max-width:720px`）恢复为单列堆叠，搜索框在标题下方、口音切换上方
- 搜索框样式：
  - 背景 `rgba(255,255,255,.06)`，边框 `1px solid rgba(255,255,255,.12)`，圆角 `10px`
  - 左侧嵌入搜索 SVG 图标，右侧清除按钮（X）
  - 聚焦时边框变为 `--cyan`，添加微光晕
  - 输入框内文字颜色 `var(--text)`，placeholder 颜色 `var(--muted)`
- 搜索按钮（或 Enter 键）触发查询；输入为空时按 Enter 清除结果

### 5. 结果展示面板

**文件**：`index.html` `<style>` + `<body>` 结构 + `<script>`

**内容**：
- 在 `topbar` 与 `#chart` 之间插入 `#search-result` 容器，默认 `display:none`
- 查询成功后展开面板，包含：
  - **单词标题行**：查询单词 + 完整音标文本（如 `/həˈləʊ/`）+ 播放按钮（播放 API 返回的单词音频或本地单词音频）
  - **音标分解条**：将 `tokens` 渲染为 pill/badge 形状的元素，每个 pill 背景色为其所属分组的颜色，文字为音标符号
  - **分组标签**：在分解条上方或下方显示分组说明（如 "短元音 · 辅音 · 双元音"）
- 面板展开动画：`max-height` + `opacity` 过渡，时长 260ms
- 交互：
  - 点击任意音标 pill → 播放对应音标发音（复用现有 `play()` 逻辑）
  - 悬停音标 pill → 对应卡片在表格中脉冲闪烁一次
  - 面板右上角关闭按钮 → 关闭面板并清除高亮

### 6. 高亮系统

**文件**：`index.html` `<style>` + `<script>`

**内容**：
- CSS 新增：
  - `.ipa-card.highlighted`：边框变 `--green`，绿色光晕 `box-shadow: 0 0 28px rgba(53,240,172,.28)`，背景微调为偏绿渐变
  - `.ipa-card.dimmed`：透明度降至 `0.35`，添加 `grayscale(0.6)`
- `highlightCards(tokenIds)` 函数：
  1. 遍历所有 `.ipa-card`，有 `data-ipa-id` 在 `tokenIds` 中的添加 `.highlighted`、移除 `.dimmed`
  2. 不在集合中的卡片添加 `.dimmed`、移除 `.highlighted`
  3. 第一个高亮卡片执行 `scrollIntoView({behavior:'smooth', block:'center'})`
- `clearHighlight()` 函数：移除所有卡片的 `.highlighted` 和 `.dimmed`，清空 `state.highlightedIds`
- 新查询前、点击清除按钮、点击关闭按钮、输入框为空时，自动调用 `clearHighlight()`

### 7. UK/US 切换联动

**文件**：`index.html` `<script>` 中（修改现有事件监听）

**内容**：
- 扩展 `state` 对象，新增 `lastSearchData: null`
- 在 `accent-toggle` 点击事件中，完成原有切换逻辑后：
  - 若 `state.lastSearchData` 存在，调用 `updateSearchResult(state.lastSearchData)`
  - 该函数会基于新口音重新执行 `extractPhonetic` + `parseIpaString` + 渲染面板 + 高亮卡片
  - 保证用户切换口音后，结果面板和高亮状态实时同步

### 8. 加载与错误状态

**文件**：`index.html` `<style>` + `<script>`

**内容**：
- 加载状态：搜索按钮显示旋转动画或文字变为 "查询中..."
- 错误提示：在搜索框下方以红色小字（`#ff6b6b`）显示错误信息，3 秒后自动淡出
- 常见错误覆盖：空输入、非英文字符、单词不存在、网络超时、API 返回无音标文本

## Assumptions & Decisions

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 音标数据来源 | `dictionaryapi.dev` 免费 API | 支持 CORS，无需后端，无需 API Key，适合纯前端静态页面 |
| US/UK 音标区分 | API 返回后按内容特征自动判断 + fallback | API 本身不严格标注 UK/US，但通过符号特征（`əʊ` vs `oʊ`）可高度准确区分 |
| 音标符号匹配策略 | 贪婪匹配（多字符优先） | 避免 `əʊ` 被拆成 `ə` + `ʊ`，`tʃ` 被拆成 `t` + `ʃ` 等错误 |
| 颜色分组数量 | 6 组 | 覆盖所有 44 个音标，每组含义明确，颜色数量适中不混乱 |
| 高亮视觉效果 | 高亮卡片绿色光晕 + 非高亮卡片暗淡 | 形成强烈视觉聚焦，引导用户关注目标音标 |
| 面板位置 | topbar 下方、chart 上方 | 符合用户从上到下阅读习惯，与搜索操作紧邻 |
| 是否引入构建工具/框架 | 否 | 保持现有零依赖单文件结构，降低维护成本 |

## Verification Steps

1. **功能验证**：输入 `hello`，面板显示 `/həˈləʊ/`，分解条显示 `h` `ə` `l` `əʊ` 四个 pill（`ə` 为短元音 cyan，`əʊ` 为双元音 green，`h` 为清辅音 red，`l` 为鼻音/近音 purple），下方表格中对应 4 张卡片高亮绿色光晕，其余卡片暗淡
2. **口音联动**：查询 `hello` 后切换 US，面板音标变为 `/həˈloʊ/`，分解条变为 `h` `ə` `l` `oʊ`（`oʊ` 经映射后对应 `əʊ`，仍显示为双元音 green），高亮状态同步更新
3. **多字符音标**：输入 `chip`，确保解析出 `tʃ` 而非 `t` + `ʃ`；输入 `boy`，确保解析出 `ɔɪ` 而非 `ɔ` + `ɪ`
4. **错误处理**：输入不存在的单词 `xyzabc`，显示 "未找到该单词"；断网时显示 "查询失败，请稍后重试"
5. **移动端**：在宽度 375px 设备上，搜索框、面板、高亮效果均正常显示，布局无溢出
