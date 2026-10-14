# 专辑详情页：曲目删除按钮与位深/采样率展示 - 独立审查

- [x] CP-R1: 专辑曲目行收藏按钮后存在删除按钮（桌面常显/移动端更多操作面板内，顺序一致）
  - **Type**: `rule`
  - **Covers**: AC-1 / TR-4.1
  - **Evidence**: LibraryAlbum.vue#L121-L129（按钮位于收藏 L112-L120 之后、同 MobileRowActions 插槽，垃圾桶 SVG、title="删除"、icon-action-btn danger）；MobileRowActions.vue#L23-L29 窄屏收进浮层，DOM 顺序即面板顺序。

- [x] CP-R2: 删除必须经 danger 二次确认，取消/遮罩/ESC 不产生请求
  - **Type**: `rule`
  - **Covers**: AC-2 / TR-4.2
  - **Evidence**: appConfirm danger（文案含歌名+物理删除不可恢复，LibraryAlbum.vue#L465-L473），cancelAppDialog settle(false)（appDialog.js#L132-L137），`if (!ok) return` 先于任何网络调用；App.vue 全局 ConfirmModal 红钮已核实。

- [x] CP-R3: 确认后物理删除、当前页刷新、总数更新、失败保留行
  - **Type**: `rule`
  - **Covers**: AC-3 / TR-4.3
  - **Evidence**: 实测 POST /api/library/delete-files `{"deleted":1,"failed":0}`，磁盘文件消失，专辑 total 2→1；前端成功后 loadAlbum 重拉（末页删空回退 page 由 watcher 单次触发，无重复加载），failed/catch 两分支 toast 且不移除行，deleteBusy 入口拦截 + 按钮 disabled。

- [x] CP-R4: 后端三条列表路径输出 sampleRate/bitsPerSample，旧 SQLite 安全回退，零 DDL/零 I/O
  - **Type**: `rule`
  - **Covers**: AC-4 / TR-1.1~1.3
  - **Evidence**: json_extract CAST 列（libraryCache.js#L202-L221、L849-L898），rowToQueryFile#L773-L774、CACHE_LIST_FIELDS#L191-L195；better-sqlite3 实测 44100/16、48000/24、缺失→0；allRowsWithMetaSpec 仅匹配 json_extract 报错时降级并记忆；无 ALTER/无 readMeta 调用；where 全参数化、order 白名单，无注入面。

- [x] CP-R5: 标签行规格格式与边界正确，字段缺失省略
  - **Type**: `rule`
  - **Covers**: AC-5 / TR-3.1
  - **Evidence**: 审查方独立 Node 断言：(44100,16)→`16bit · 44.1kHz`、(48000,24)→`24bit · 48kHz`、(44100,0)→`44.1kHz`、(0,0)→`''`；异常字符串/null/负数均省略；规格位于格式后时长前。

- [x] CP-R6: 其他调用页输出零变化
  - **Type**: `rule`
  - **Covers**: AC-6 / TR-3.2
  - **Evidence**: 6 处既有 formatTrackTags 调用（AddToPlaylistModal、LibraryGenre、Library×2、LibraryPlaylists、LibrarySearch×2）均不传第二参数；唯一启用点 LibraryAlbum.vue#L99；trackToSnapshot 未改，新字段不进收藏/歌单快照。

- [x] CP-U1: 交互与视觉一致性
  - **Type**: `rubric`
  - **Covers**: AC-7 / TR-4.5
  - **Scale**: 1-5
  - **Anchors**: 1 = 样式突兀/布局错位；3 = 可用但有可见差异；5 = 与既有控件完全融合
  - **Pass Threshold**: >= 4
  - **Evidence**: 评分 4/5。复用全局 icon-action-btn（34px/9px 圆角/danger hover 变红）与同一 .track-tags（移动端既有 display:none 未动）；扣 1 分因未取得桌面/移动实拍截图，留待用户本地目检（非代码缺陷）。

- [x] CP-R7: 前端构建通过
  - **Type**: `rule`
  - **Covers**: AC-8 / TR-5.1
  - **Evidence**: 实现方与审查方分别执行 `npm run build`，均退出码 0（✓ built in ~1.4s）。

## Findings（均为 advisory/low，不阻塞）
1. 删除后页面 toast 与 WS 全局横幅「移除 1 首歌曲」双重提示——与 FileManager 既有行为一致；后续可对 reason='manual-delete' 抑制横幅。
2. 确认框点确认即关闭，慢网下无「处理中」文案（busy 防重本身有效，与全应用 appConfirm 模式一致）。
3. 删除正在播放的曲目时当前音频播到曲终才切下一首（pruneQueueLocalTracks 只调索引），不报错；spec 已委托现有机制。
4. formatAudioSpec 对非整数位深/22050Hz 舍入为防御性边界，后端 CAST INTEGER 已保证输入。
5. session slim 每曲 +2 数值字段，8000 首上限增量约 100–250KB，触配额概率低。
6. json_extract 降级探测与同文件迁移回填处的探测逻辑重复（状态不共享），纯整洁性建议。

## Review History

### Review R1
- **Result**: `pass`
- **Evidence**: 独立审查员（fresh context，只读）通读 spec.md/tasks.md 与提交 26f63a1 全部 diff；独立复跑 npm run build、format 断言与 better-sqlite3 查询验证；逐条核对 AC-1~AC-8 及全部 formatTrackTags 调用点、SQL 参数化、删除/分页/WS 链路；无 actionable 发现，6 条 advisory 均不阻塞。UI 实拍目检由用户在本地验收（CP-U1 已按代码走查给 4/5 过线）。
