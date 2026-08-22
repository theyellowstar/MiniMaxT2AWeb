# MiniMax T2A 语音合成工作台

单文件网页工具（`index.html`），整合 MiniMax 语音合成：同步/异步合成、任务查询、tar 解包、字幕（播放器同步 + SRT 转换）、音色管理与试听、历史记录。**零依赖、零构建**，双击 `index.html` 即可用（也可 `python -m http.server` 起服务）。

## 硬性约束

- **保持单文件、无外部依赖**：不引 CDN、不加构建步骤。纯 JS zip/tar 解析器均为手写。
- **每次修改后必须跑语法校验**（见文末「验证」），历史上有多次 shell 转义导致误判的教训——校验失败先确认是代码问题还是校验脚本问题。
- **用户偏好极简 UI**：按钮能少则少。曾明确要求删除：复制 task_id 按钮、原始响应调试按钮/弹窗、单独的 SRT 下载按钮、字幕静态预览、自动轮询复选框（并入轮询间隔下拉）、字幕开关复选框（异步接口无效）。
- **不要重新引入已被验证为错误的假设**（见「死代码黑名单」）。

## 代码结构（index.html 内 JS 分区，按出现顺序）

1. 工具函数：`$`/`$$`/`esc`/`uuid`/`fmtBytes`/`fmtDur`/`timeStr`/`fileTs`/`MIME`/`TASK_STATUS`/`toast`/`copyText`/`hexToBytes`/`downloadBlob`/`saveBlob`
2. 持久化：`settings`（localStorage）、`history`（localStorage）、IndexedDB（`idbOpen/idbPut/idbGet/idbDel/idbClear`）
3. API 层：`apiCall(path, {method, body, formData})`（Bearer 鉴权、错误码映射 `CODE_MSG`）、`fetchBlob`
4. tar 解析（ustar，魔数校验 `looksLikeTar`）、`isSubtitleName`、SRT 工具（`fmtSrtTs`/`titlesToSrt`）、`parseSubtitle`（多字段名兼容）
5. 请求体构造：`buildCommonBody`（同步带 `subtitle_enable:true`；异步带 `text_file_id` 或 `text`）
6. 合成流程：`runSync` / `runAsyncCreate` / `queryTaskById` / `retrieveById`（含 tar 解包分发）
7. 字幕下载：`downloadSubtitle`（转 SRT）/ `downloadSubtitleRaw`（原文件）
8. 附件：`downloadExtra`；轮询：`setupTicker`/`tick`；手动查询/下载：`manualQuery`/`manualRetrieve`
9. 文本文件上传：`uploadTextFile`（超大 txt 自动拆分打包 zip）、`splitToZipDownload`、zip 构建（`crc32`/`buildZip`/`splitChapters`）
10. 播放器：`playEntry`/`playBlobInPlayer`/`setPlayerSubs`/`showCurrentSub`（`timeupdate` 驱动同步字幕）
11. 结果卡片：`renderSyncResult`/`renderAsyncResult`/`renderAsyncResultIfActive`
12. 历史渲染：`historyItemHtml`/`actsFor`/`renderHistory` + 筛选（`filters`）
13. 表单联动：`syncFormFromSettings`/`syncSettingsFromForm`/`updateFileModeUI`
14. 音色：`fetchVoices(silent)`/`renderVoiceBank`/`previewVoice`/`recordVoiceUse`（最近使用）
15. 设置弹窗、导出/导入、事件绑定（`bindEvents`）、初始化

**新增一个按钮的标准路径**：在 `actsFor`（历史条目）或结果卡片模板里加 `<button data-act="xxx" data-id=...>`，再在 `bindEvents` 的两处事件委托（`#historyList` 用 `act ===`、结果卡片用 `btn.dataset.act ===`）加分支。

## 已验证的 MiniMax API 事实（勿再猜错）

### 异步流程（国内 base `https://api.minimaxi.com`，国际 `https://api.minimax.io`）
1. 创建：`POST /v1/t2a_async_v2`，body：`{model, text | text_file_id, voice_setting{voice_id,speed,vol,pitch}, audio_setting{audio_sample_rate,bitrate,format,channel}, language_boost?}` → `{task_id, file_id, usage_characters}`
2. 查询：`GET /v1/query/t2a_async_query_v2?task_id=` → `{status: Processing|Success|Failed|Expired, file_id}`（限 10 次/秒；**无任何字幕字段**）
3. 检索：`GET /v1/files/retrieve?file_id=` → `{file: {filename: "<id>.tar", purpose: "t2a_async", download_url, bytes: 0}}`，**下载链接 9 小时有效**
4. **关键：该版本异步接口始终返回 `with_meta.tar` 归档**，与 `subtitle_enable` 无关（该参数在异步接口被忽略）。tar 内结构（实测）：
   - `content-<id>.mp3`（音频）
   - `content-<id>.titles` —— JSON 数组：`{text, pronounce_text, time_begin, time_end(毫秒), text_begin, text_end, ...}`，句级时间戳
   - `content-<id>.extra` —— `{audio_length(ms), audio_sample_rate, audio_size, bitrate, vol, word_count}`
   - 文件名带目录前缀（展示取 basename）

### 同步流程
- `POST /v1/t2a_v2`，body：`{model, text, stream:false, subtitle_enable:true, output_format: hex|url, voice_setting, audio_setting{sample_rate,bitrate,format,channel}}`
- 响应：`data.audio`（hex 或 URL）、`data.subtitle_file`（**subtitle_enable=true 时**返回字幕下载链接，仅非流式有效）、`extra_info{audio_length,audio_sample_rate,audio_size,usage_characters}`
- 字幕链接先直连 fetch，失败带 Bearer 头重试

### 文件上传与限制
- `POST /v1/files/upload`（multipart：`file` + `purpose=t2a_async_input`）→ `file_id`
- **`text_file_id` 必须是整数（int64）**：以字符串发送会报 2013 `invalid params`（本会话曾因此误判为「超限」「zip 不支持」；修复：`buildCommonBody` 用 `+settings.textFileId`）。官方单 txt 文件上限 <100 万字符（不是 10 万）。zip 章节包内须同一格式 txt/json（json 字段 `title/content/extra`，可只给 `content`）；文件名必须 `1.json`/`2.json`…（纯数字，无前缀/补零）——`chapter_001.json` 会报 `check file in zip error`。拆分阈值 `SPLIT_CHARS=20000`：超过 2 万字符自动拆 zip（每章 ≤2 万），这是用户明确偏好，勿改回「单文件优先」。
- `GET/POST /v1/get_voice`，body `{voice_type:"all"}` → `system_voice`/`voice_cloning`/`voice_generation` 数组（复刻/文生音色需成功合成过一次才出现）

### 错误码（CODE_MSG）
1004 鉴权失败 · 1002 限流 · 1008 余额不足 · 1039 TPM 限流 · 1042 非法字符 >10% · 2013 参数错误。apiCall 会同时展示服务端 `status_msg`。

## 存储 Schema

**localStorage**：
- `mm_tts_settings_v1`：`{apiKey, baseUrl, customBase, groupId, proxy, voiceId, model, speed, vol, pitch, sampleRate, bitrate, format, channel, languageBoost, outputFormat, emotion, useTextFile, textFileId, textFileName, textFileTs, saveMethod, pollInterval(0=不轮询)}`
- `mm_tts_history_v1`：历史数组（最多 200 条）。条目字段：`{id, mode:'sync'|'async', ts, text, voiceId, model, format, speed, vol, pitch, status, errMsg, extra{durationMs,size,usage,sampleRate}, taskId, fileId, taskStatus, usage, downloadUrl, fileName, urlExpiresAt, subtitleUrl, subtitleFileName, subtitleCached, tarEntries[], extraFiles[{filename,size,cached,idx}], textFileId, manual, lastErr}`。`raws` 为历史遗留字段，saveHistory 空间不足时会自动剥离
- `mm_tts_voices_v1`：`{ts, groups:{system_voice,voice_cloning,voice_generation:[{id,name}]}}`
- `mm_tts_voice_recent_v1`：`[{id,name,ts}]` 最近使用（≤10）

**IndexedDB**（库 `mm_tts_db`，store `audio`）键约定：
- `<entry.id>` → 解包后的音频 Blob
- `sub:<id>` → 字幕 Blob（.titles/同步字幕）
- `exf:<id>:<i>` → tar 内附件
- `pv:<voiceId>:<model>` → 试听缓存（同一音色+模型只计费一次）

删除历史条目时按上述键一并清理（见 `delHistory`）。

## 关键实现细节 / 坑

- **SRT 输出**：`titlesToSrt` 转 `HH:MM:SS,mmm --> HH:MM:SS,mmm`，**UTF-8 必须带 BOM**（文件里那个 `'﻿'` 字符不能丢，否则中文播放器乱码）；文件名与音频同基名（`entryAudioBase`）
- **saveBlob（防 Windows MOTW）**：`showSaveFilePicker` 写入的文件无 Zone.Identifier。需**用户手势**（transient activation ~5 秒）——音频与 SRT 链式保存用 `setTimeout(400ms)` 保证手势未过期；不支持/异常时回退 `downloadBlob`（会有网络标记）。返回值 `'picker'|'classic'|'cancelled'`
- **播放器同步字幕**：`playEntry` 加载 `sub:<id>` 解析后交给 `player.subs`，`timeupdate` 事件里 `showCurrentSub` 按毫秒匹配 `[begin, end)` 显示当前句。勿改回静态预览（用户明确否掉）
- **fetchVoices 自动化**：页面加载时（有 Key 且无缓存/超 7 天）静默拉取；设置保存时 Key 变化自动拉取。`silent` 参数控制错误提示
- **轮询**：`pollInterval=0` 表示不轮询（含创建后立即查一次也跳过）；`setupTicker` 在设置变更时重建
- **textarea 的 placeholder 被 `updateFileModeUI` 动态改写**，改原文提示时记得同步那里
- **zip 拆分上传**：`buildZip` 用原生 `CompressionStream('deflate-raw')` 做 DEFLATE（不可用回退 STORE）并写入有效 DOS 时间戳；章节写成 JSON `{content:文本}`，**文件名必须是 `1.json`/`2.json`…（纯数字）**——`chapter_001.json` 会报 `check file in zip error`
- 头部/手动操作区（task_id 查询、file_id 下载）在异步模式显示

## 死代码黑名单（验证过的错误假设，勿恢复）

- ❌ 查询响应会返回 `subtitle_file_id` 等字幕字段 → 实际没有，字幕在 tar 里
- ❌ `files/retrieve` 会返回多文件数组 / 字幕对象 → 实际单文件（tar）
- ❌ 查询会返回 file_id 数组（zip 多章节）→ 实际单 file_id
- ❌ `text_file_id` 传字符串也能被接受 → 实际必须整数(int64)，字符串报 2013 `invalid params`（这是 zip/单 txt 创建任务失败的真正根因，非 zip 格式问题）
- ❌ 静态字幕预览（历史折叠区/卡片预览）→ 已改为播放器同步字幕
- ❌ 独立 SRT 下载按钮 / 复制 ID 按钮 / 原始响应调试按钮
- ❌ `autoPoll` 复选框、异步 `subtitleEnable` 复选框（参数无效）

## 验证方法

每次修改后运行（Windows Git Bash）：

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*?)<\/script>/g);
const code=m[m.length-1].replace(/^<script>|<\/script>$/g,'');
new Function(code); console.log('JS syntax OK');
"
```

功能自测时注意：`eval` 中 `const/let` 不泄漏到外层作用域，提取函数测试需用 `function` 声明；shell 引号转义易出错，优先用 Grep 工具核对字符串。
