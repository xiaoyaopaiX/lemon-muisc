# 专辑详情页：曲目删除按钮与位深/采样率展示 - 实施计划

## Task 1: 后端曲库缓存输出 sampleRate / bitsPerSample
- **Status**: `completed`
- **Completion Evidence**:
  - TR-1.1（代码走查）：`libraryCache.js` 分页查询（queryCachedTracks）与 by-paths 查询 SELECT 增加 `CAST(json_extract(meta_json,'$.sampleRate') AS INTEGER) AS sample_rate` 等两列，`rowToQueryFile` 输出 `sampleRate/bitsPerSample`；`CACHE_LIST_FIELDS` 追加两字段使 getAllCachedTracks（all=1）/rowToSlimFile 同源输出；getTrackById 走 rowToFile 全量 meta 天然包含。
  - TR-1.2（代码走查）：新增 `allRowsWithMetaSpec` 惰性探测，"no such function: json_extract" 时自动以无规格列 SQL 重试且模块级缓存降级状态；rowToQueryFile 对缺失列以 `Number(row.sample_rate)||0` 输出 0；slim 路径无值时省略键，前端归一化为 0。
  - TR-1.3（diff 审查）：未新增数据库列（无 ALTER/DDL 变更）、未引入 readMetaLite/parseFile 调用，零额外文件 I/O。
  - `node --check server/utils/libraryCache.js` → SYNTAX_OK。SQL 实测待 npm ci 完成后在 Task 5 补充。
- **Priority**: high
- **Depends On**: None
- **Description**:
  - 在 `server/utils/libraryCache.js` 中让曲目列表查询输出 `sampleRate`、`bitsPerSample`，数据取自已缓存的 `meta_json`（扫描时 `readMetaLite` 已写入），不做表结构变更、不增加磁盘 I/O。
  - `queryCachedTracks` 的分页 SELECT 与 `queryTracksByPaths` 的 SELECT 增加 `CAST(json_extract(meta_json, '$.sampleRate') AS INTEGER)` / `$.bitsPerSample` 输出列；`rowToQueryFile` 读取并输出（缺失为 0）。
  - `CACHE_LIST_FIELDS` 白名单追加 `'sampleRate'`、`'bitsPerSample'`，使 `all=1`（`getAllCachedTracks` → `rowToSlimFile`）同源输出。
  - 兼容性：对 `json_extract` 不可用的旧 SQLite 做安全回退（查询失败时以不含规格列的语句重试一次，规格字段按 0/缺省输出），保证接口始终 200。
- **Acceptance Criteria Addressed**: AC-4
- **Test Requirements**:
  - `rule` TR-1.1: `queryCachedTracks`、`queryTracksByPaths`、`getAllCachedTracks` 三条路径返回的条目包含数值型 `sampleRate`、`bitsPerSample` 键；证据：代码走查 + 对含 FLAC 曲库的接口实测（`GET /api/library/tracks?limit=...`、`all=1`、by-paths）。
  - `rule` TR-1.2: meta_json 无该字段或 SQLite 无 json_extract 时接口不抛错（值为 0/缺省）；证据：代码走查回退分支 + 构造无字段行查询验证。
  - `rule` TR-1.3: 不新增数据库列、不调用 `readMetaLite`/`parseFile` 等文件 I/O；证据：diff 审查。
- **Notes**: `getTrackById` 走 `rowToFile` 全量 meta，天然含两字段，无需改动；`extractSearchFields`/表列无需改动（不参与筛选排序）。

## Task 2: 前端曲库映射透传规格字段
- **Status**: `completed`
- **Completion Evidence**:
  - TR-2.1（代码走查）：`fileToLibraryTrack` 返回对象新增 `sampleRate: Number(file.sampleRate)||0`、`bitsPerSample: Number(file.bitsPerSample)||0`；fetchLibraryTracksPage 经此映射，专辑页 track 可直接读取。
  - TR-2.2（代码走查）：`saveSessionTracks` slim 快照追加两字段；`loadSessionTracks` 经 fileToLibraryTrack 恢复（同名键直读），刷新后不丢失。
  - 未改动 trackToSnapshot（收藏/歌单快照），符合 Notes 约定。
- **Priority**: high
- **Depends On**: Task 1
- **Description**:
  - `src/stores/library.js` 的 `fileToLibraryTrack` 返回对象追加 `sampleRate: Number(file.sampleRate) || 0`、`bitsPerSample: Number(file.bitsPerSample) || 0`。
  - `saveSessionTracks` 的 slim 快照追加这两个字段，`loadSessionTracks` 经 `fileToLibraryTrack` 恢复时不丢失（仅缓存瘦身用途，不改变既有字段）。
- **Acceptance Criteria Addressed**: AC-4、AC-5
- **Test Requirements**:
  - `rule` TR-2.1: 专辑页 `fetchLibraryTracksPage` 得到的 track 对象可读出 `sampleRate/bitsPerSample` 数值；证据：代码走查 + 浏览器控制台/响应比对。
  - `rule` TR-2.2: sessionStorage slim 快照含新字段且刷新恢复后仍在；证据：代码走查。
- **Notes**: `trackToSnapshot`（收藏/歌单快照）不要求携带规格，保持现状。

## Task 3: 标签格式化支持位深/采样率（默认关闭）
- **Status**: `completed`
- **Completion Evidence**:
  - TR-3.1：Node ESM 断言全部 PASS——(44100,16)→`16bit · 44.1kHz`；(48000,24)→`24bit · 48kHz`；(44100,0)→`44.1kHz`；(0,0)→`''`；另测 (96000,24)→`24bit · 96kHz`；formatTrackTags(base,{audioSpec:true})→`跨时代 · FLAC · 16bit · 44.1kHz · 4:23`。
  - TR-3.2：默认调用与传 `{}` 均输出 `跨时代 · FLAC · 4:23`（断言 PASS）；既有 6 处调用点未传第二参数，输出零变化。
- **Priority**: high
- **Depends On**: None
- **Description**:
  - 在 `src/utils/format.js` 新增导出 `formatAudioSpec(track)`：返回 `{ bits, hz }` 组装的规格段——位深 `bitsPerSample>0` 时输出 `${n}bit`；采样率 `sampleRate>0` 时按 kHz 输出（整数 kHz 无小数，如 `48kHz`；否则一位小数，如 `44.1kHz`）；以 ` · ` 连接，两者皆空返回空串。
  - 扩展 `formatTrackTags(track, options = {})`：仅当 `options.audioSpec` 为真时，在格式（format）之后、时长（duration）之前插入规格段；默认行为完全不变。
- **Acceptance Criteria Addressed**: AC-5、AC-6
- **Test Requirements**:
  - `rule` TR-3.1: `formatAudioSpec`/`formatTrackTags` 对 `(sampleRate=44100,bits=16)` → `16bit · 44.1kHz`；`(48000,24)` → `24bit · 48kHz`；`(44100,0)` → `44.1kHz`；`(0,0)` → `''` 且标签行与原输出一致；证据：用 Node 对格式化函数做快速断言（或手工走查四组用例）。
  - `rule` TR-3.2: 不传第二参数的既有调用点输出零变化；证据：全局调用点比对（6 处既有调用不传选项）。

## Task 4: 专辑页接入规格展示与删除按钮
- **Status**: `in_progress`
- **Priority**: high
- **Depends On**: Task 2、Task 3
- **Description**:
  - `src/views/LibraryAlbum.vue`：
    - 标签行改为 `formatTrackTags(song, { audioSpec: true })`。
    - 在 `MobileRowActions` 插槽内收藏按钮之后新增删除按钮：垃圾桶 SVG（与文件管理器删除图标风格一致的线性图标，16px，stroke=currentColor），`class="icon-action-btn"`，`title="删除"`，`@click.stop` 触发确认；新增 deleting key 状态防止并发。
    - 引入 `ConfirmModal`：`danger`、标题/文案含曲目名与「将从磁盘物理删除，不可恢复」提示、`busy` 期间禁用；取消/ESC/遮罩关闭不删除。
    - 确认后调用 `api.library.deleteFiles([getTrackFilePath(song)])`：成功→关闭弹窗、toast 成功（含歌名）、重新执行 `loadAlbum()`（保持当前页，使列表补位、`trackCount` 更新为服务端 total）；接口返回 `failed` 或抛错→toast 错误且保留行；移动端同时关闭该行操作面板。
- **Acceptance Criteria Addressed**: AC-1、AC-2、AC-3、AC-5、AC-6、AC-7
- **Test Requirements**:
  - `rule` TR-4.1: 删除按钮在模板中位于收藏按钮之后、`MobileRowActions` 插槽内，含 `title="删除"` 与垃圾桶图标；证据：代码走查 + 桌面/移动视口截图。
  - `rule` TR-4.2: 删除请求只在确认后发出，且体为 `{ filePaths: [绝对路径] }`；取消路径无请求；证据：浏览器 Network 面板交互验证。
  - `rule` TR-4.3: 删除成功后 `loadAlbum()` 被重新调用、被删行消失、总数减 1；失败有错误 toast 且行保留；busy 期间不可重复确认；证据：浏览器交互（测试曲目）+ 代码走查。
  - `rule` TR-4.4: 标签行渲染 `… · FLAC · 16bit · 44.1kHz · 4:23` 形态，规格缺失的曲目不错误显示 0；证据：浏览器截图/走查。
  - `rubric` TR-4.5: 视觉一致性；scale 1-5；anchors 1=样式突兀/布局错位/标签换行，3=可用但有可见视觉差异，5=与既有图标按钮及标签行完全融合、hover/间距/超长省略/移动端面板行为一致；threshold >= 4；证据：桌面 1280px 与移动 390px 视口截图对比。

## Task 5: 构建与端到端验证
- **Status**: `pending`
- **Priority**: medium
- **Depends On**: Task 4
- **Description**:
  - 执行 `npm run build` 确认前端构建零错误。
  - 启动应用（dev/server），在含本地音频的专辑详情页实测：标签规格展示、删除确认→删除→列表刷新全链路；验证其他页面（音乐库首页/搜索）标签文案未变化。
  - 若环境无曲库数据无法完整实测删除，则以构建 + 代码走查 + 静态页面渲染验证作为证据，并在 Completion Evidence 中如实标注受限项。
- **Acceptance Criteria Addressed**: AC-2、AC-3、AC-4、AC-5、AC-6、AC-7、AC-8
- **Test Requirements**:
  - `rule` TR-5.1: `npm run build` 退出码 0；证据：命令输出。
  - `rule` TR-5.2: 专辑页接口响应含 sampleRate/bitsPerSample，UI 端到端行为符合 AC；证据：curl/Network 面板与浏览器截图。
  - `rubric` TR-5.3: 整体回归观感；scale 1-5；anchors 1=其他页面出现回归/错乱，3=主流程可用但存在小瑕疵，5=主流程顺畅且无回归；threshold >= 4；证据：浏览器走查记录。
