# 专辑详情页：曲目删除按钮与位深/采样率展示 - 产品需求文档

## Overview
- **Summary**: 在音乐库专辑详情页（`/library/album`，`LibraryAlbum.vue`）每首曲目行的收藏（☆）按钮之后新增「删除」按钮；并在曲目副标题标签行（专辑 · 格式 · 时长）中展示该曲目的位深与采样率。
- **Purpose**: 让用户在浏览专辑时可以直接剔除不需要的本地曲目，无需跳转文件管理器；同时在曲目列表中直接辨别无损规格（如 16bit/44.1kHz、24bit/96kHz），无需逐个打开「歌曲信息」弹窗。
- **Target Users**: 在 NAS/本地音乐库中整理专辑、关注音频规格的用户。

## Goals
- 专辑详情页曲目行：收藏按钮后出现删除按钮，点击后经二次确认物理删除该本地音频文件，列表即时刷新。
- 专辑详情页曲目标签行：在格式（如 FLAC）之后、时长之前，按「位深 · 采样率」顺序展示（如 `16bit · 44.1kHz`）；字段缺失时对应项省略，不错误显示 0。
- 位深/采样率数据来自现有曲库扫描缓存，不引入额外的逐文件实时读取。

## Non-Goals
- 不在音乐库首页、歌手页、流派页、歌单页、搜索结果页、「加入歌单」弹窗等其他复用 `formatTrackTags` 的位置新增位深/采样率展示（保持现状）。
- 不新增批量删除、不新增「删除整张专辑」能力（仅逐曲删除）。
- 不做删除到回收站/软删除；复用现有物理删除语义（不可恢复）。
- 不对历史缓存做全量强制重扫迁移；缺失规格字段的旧缓存记录在下次正常扫描（文件 mtime/size 变化或手动刷新扫描）后自然补齐。
- 不改动「歌曲信息」弹窗（`SongInfoModal.vue`）中已有的采样率/位深展示。
- 不处理 DSD 特殊命名（如 DSD64/2.8 MHz），采样率统一以 kHz 数值呈现。

## Background & Context
- 专辑详情页曲目行操作区由 `src/views/LibraryAlbum.vue` 中的 `MobileRowActions` 插槽构成，现有按钮顺序：加入试听列表（+）、加入歌单（列表图标）、收藏（☆）。用户截图在 ☆ 后红圈标注删除按钮位置，在标签行（如「跨时代 · FLAC · 4:23」）红框标注规格展示位置。
- 标签行文案由 `src/utils/format.js` 的 `formatTrackTags(track)` 生成，被 `LibraryAlbum.vue`、`Library.vue`、`LibraryArtist` 相关页、`LibraryGenre.vue`、`LibraryPlaylists.vue`、`LibrarySearch.vue`、`AddToPlaylistModal.vue` 共用，故必须以选项方式仅对专辑详情页开启新字段。
- 后端扫描（`server/meta.js` 的 `readMetaLite` → `buildMetaFromParsed`）始终返回 `sampleRate`、`bitsPerSample`；`server/utils/libraryCache.js` 的 `upsertCacheEntry` 将完整 lite meta 序列化写入 `library_index.meta_json`。但：
  - 分页查询 `queryCachedTracks` 仅 SELECT 固定列并经 `rowToQueryFile` 输出，不含这两个字段；
  - 全量/路径查询走 `rowToSlimFile`，受白名单 `CACHE_LIST_FIELDS` 限制也不输出。
- 删除链路已存在：`POST /api/library/delete-files`（校验路径必须在音乐库/下载目录内，支持文件已不存在的幽灵记录）→ `removeCachePaths` 清索引 → WS 广播 `library:removed`；前端 `api.library.deleteFiles(paths)` 已封装，`stores/library.js` 已监听该事件清理 `libraryTracks`、收藏/最近/歌单引用、播放队列并递减侧栏角标。
- 二次确认可复用 `src/components/ConfirmModal.vue`（支持 `danger` 红色确认按钮与 `busy` 状态）。
- 移动端（≤768px）现有 CSS 对专辑页 `.track-tags { display: none; }`，标签行本就不在移动端展示；删除按钮则收进「更多操作」弹出面板，与其他操作按钮一致。

## Functional Requirements
- **FR-1（删除入口）**: 专辑详情页每个曲目行的收藏按钮之后渲染一个删除图标按钮（垃圾桶图标），`title` 为「删除」；桌面端常显，移动端出现在「更多操作」展开面板内，且排在收藏项之后。
- **FR-2（删除确认）**: 点击删除按钮弹出危险操作确认框（`ConfirmModal` danger 样式），文案包含曲目名并提示文件将被物理删除、不可恢复，展示操作中的 busy 状态；点击取消或遮罩/ESC 关闭时不产生任何删除调用。
- **FR-3（执行删除与刷新）**: 确认后以该曲文件路径调用 `POST /api/library/delete-files`；成功后重新拉取当前专辑页（保持当前分页），被删曲目从列表消失、专辑曲目总数正确；失败时展示错误提示且曲目保留。
- **FR-4（后端字段输出）**: `GET /api/library/tracks` 分页响应的每个曲目对象包含数值型 `sampleRate`（Hz）与 `bitsPerSample`（bit），数据来自已缓存的 `meta_json`；无法提供时为 0/缺省，不报错。同源的全量（`all=1`）与 by-paths 查询保持字段一致。
- **FR-5（前端字段透传）**: 前端曲库映射 `fileToLibraryTrack` 保留 `sampleRate`、`bitsPerSample`，使专辑页曲目对象可直接读取。
- **FR-6（规格展示）**: 专辑详情页标签行按 `专辑 · 年份/流派（现有逻辑） · 格式 · 位深 · 采样率 · 时长` 输出；位深格式为 `{n}bit`（如 `16bit`、`24bit`），采样率格式为 kHz（整数值不带小数，如 `48kHz`；否则保留一位小数，如 `44.1kHz`、`96kHz`）；任一字段为 0/缺失则省略该项，两者都缺省时标签行与现状完全一致。
- **FR-7（影响面隔离）**: 除专辑详情页外，其他使用 `formatTrackTags` 的页面输出不发生变化。

## Non-Functional Requirements
- **NFR-1（兼容性）**: 不进行数据库表结构变更；旧版 SQLite 若不支持 `json_extract` 时不得导致列表接口报错（回退为不输出该字段）。
- **NFR-2（性能）**: 列表查询不增加逐文件 I/O，仅利用已缓存 meta；不改变现有分页响应速度量级。
- **NFR-3（安全性）**: 删除仍受既有服务端路径归属校验保护（仅限音乐库/下载目录），前端不得绕过确认直接删除。
- **NFR-4（视觉一致性）**: 删除按钮复用 `icon-action-btn` 样式与交互（hover、尺寸、间距、移动端面板行为），不破坏行布局；标签行在窄宽度下保持省略号行为，不换行撑高。
- **NFR-5（可构建性）**: 改动后前端 `npm run build` 通过，无未定义变量/语法错误。

## Constraints
- **Technical**: Vue 3（`<script setup>`）+ Pinia 风格的 composable store；Express + better-sqlite3；不引入新依赖。
- **Business**: 删除为不可逆物理删除，必须有显式确认步骤。
- **Dependencies**: 复用现有 `/library/delete-files`、`ConfirmModal`、WS `library:removed` 清理机制；规格字段依赖扫描缓存 `meta_json` 已有的数据。

## Assumptions
- 用户所说「删除」指删除服务器上的本地音频文件本身（与「文件管理器」页既有删除语义一致），而非仅从专辑视图中移除。
- 用户截图中的「详情」特指专辑详情页曲目行标签行；其他页面本次不加规格展示。
- 现有曲库中 FLAC/WAV/APE 等无损文件的缓存 meta 普遍含 `sampleRate/bitsPerSample`；MP3 通常无位深，按 FR-6 省略。
- 用户单用户音乐库内删除单首文件，播放队列/收藏中的同源引用由现有 WS 清理逻辑处理，无需专辑页额外处理。

## Acceptance Criteria

### AC-1: 专辑曲目行展示删除按钮
- **Type**: `rule`
- **Given**: 用户进入任一含曲目的专辑详情页
- **When**: 页面渲染曲目列表
- **Then**: 每一行操作区中，收藏（☆）按钮之后存在一个 `title` 为「删除」的垃圾桶图标按钮；桌面端常显，移动端在「更多操作」面板中按相同顺序出现
- **Pass Condition**: 检查 `LibraryAlbum.vue` 模板，删除按钮位于收藏按钮之后、位于 `MobileRowActions` 插槽内，图标与 `title` 符合要求
- **Evidence**: 代码审查 + 浏览器渲染截图（桌面与移动视口）

### AC-2: 删除必须二次确认且取消不删除
- **Type**: `rule`
- **Given**: 用户点击某行删除按钮
- **When**: 弹出确认框后点击「取消」、遮罩或按 ESC
- **Then**: 不发起 `/library/delete-files` 请求，列表无变化，确认框关闭
- **Pass Condition**: 删除网络调用只在确认按钮点击后发生；确认框为 danger 样式且文案含曲目名与不可恢复提示
- **Evidence**: 代码审查 + 浏览器网络面板/交互验证

### AC-3: 确认删除后文件删除且列表刷新
- **Type**: `rule`
- **Given**: 删除确认框已弹出
- **When**: 用户点击确认且服务端删除成功
- **Then**: 调用 `api.library.deleteFiles([该曲 filePath])`，随后重新加载当前专辑页；被删曲目不再显示，头部曲目总数与分页器总数减少 1；删除接口失败时曲目保留并出现错误提示
- **Pass Condition**: 删除处理函数调用路径正确、成功后重新拉取当前页数据并以返回 total 更新 `trackCount`；失败分支有 toast 且不移除行；确认过程具备 busy 防重复提交
- **Evidence**: 代码审查 + 浏览器交互（测试曲目）或删除逻辑走查

### AC-4: 后端曲目列表返回采样率与位深
- **Type**: `rule`
- **Given**: 曲库缓存中存在含规格信息的曲目
- **When**: 请求 `GET /api/library/tracks`（含专辑筛选分页，及 `all=1`、`/tracks/by-paths` 同源路径）
- **Then**: 每个曲目对象包含数值字段 `sampleRate` 与 `bitsPerSample`；缓存缺失时为 0，接口 HTTP 200 不报错；旧 SQLite 无 `json_extract` 时接口仍 200（字段缺省）
- **Pass Condition**: `queryCachedTracks`/`queryTracksByPaths`/slim 列表输出这两个字段，且 SQL 异常被安全回退
- **Evidence**: 代码审查 + 接口实测（curl/浏览器，对含 FLAC 的曲库）

### AC-5: 专辑标签行正确展示规格且字段缺失时省略
- **Type**: `rule`
- **Given**: 专辑详情页曲目携带 `format/duration/sampleRate/bitsPerSample`
- **When**: 渲染标签行
- **Then**: 规格段位于格式之后、时长之前，输出形如 `跨时代 · FLAC · 16bit · 44.1kHz · 4:23`；位深为 0 时仅省略位深、采样率为 0 时仅省略采样率、均为 0 时与现状一致；采样率整数 kHz 不显示小数（`48kHz`），非整数保留一位（`44.1kHz`）
- **Pass Condition**: 格式化函数对 `(44100,16)`、`(48000,24)`、`(44100,0)`、`(0,0)` 四组输入输出符合规则，且仅专辑页启用
- **Evidence**: 代码审查 + 浏览器渲染验证

### AC-6: 改动不影响其他列表页
- **Type**: `rule`
- **Given**: 音乐库首页、歌手/流派/歌单/搜索页及加入歌单弹窗仍以默认方式调用 `formatTrackTags`
- **When**: 渲染任一曲目标签
- **Then**: 标签文案与改动前完全一致（不含位深/采样率）
- **Pass Condition**: `formatTrackTags` 新增行为由显式选项开启，默认关闭；所有既有调用点不传该选项
- **Evidence**: 代码审查（全局调用点比对）

### AC-7: 交互与视觉质量
- **Type**: `rubric`
- **Dimension**: 删除按钮与标签行新增内容的视觉/交互一致性
- **Scale**: 1-5
- **Anchors**: 1 = 按钮样式突兀或撑乱行布局、标签换行重叠；3 = 功能可用但间距/图标/文案与现有控件有可见差异；5 = 按钮与既有图标按钮在尺寸、hover、颜色、间距上完全一致，标签行自然延展、超长省略、移动端行为正确
- **Pass Threshold**: >= 4
- **Evidence**: 桌面与移动视口下专辑页截图/走查

### AC-8: 前端构建通过
- **Type**: `rule`
- **Given**: 全部改动完成
- **When**: 执行 `npm run build`
- **Then**: 构建成功，无编译错误/未定义引用
- **Pass Condition**: 退出码 0
- **Evidence**: 构建命令输出
