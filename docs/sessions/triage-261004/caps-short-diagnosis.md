<!-- delegate-outcome: succeeded -->

# #166 CapsWriter 短音频失败边界诊断（只读）

派发：`dlg-20261004-023618-c714a0`；代码基线：`ff92a175243dde36143da867a95e2fdad384e16c`（= 本分支 HEAD，生产运行版本相同）；Verify-Mode `diagnostic-report`。

## 一句话结论

121 秒终态**已被完整解释到客户端计时边界**：该 wav 走 `GenericDownloader`，本仓从未对它 ffprobe，`media_duration=None` → 不传 `deadline_total` → SDK 自动预算取下限 `max(120, 解码时长+60)=120` 秒；服务端在该预算内没有给出 final 结果，客户端主动放弃并以 `AsrError(code=timeout, 「转录超过自动预算：远端转录阶段超时」)` 显式失败。**「服务端为什么对 5.55s 音频不给结果」仍未证**（本卡禁止向生产 ASR 提交音频），根因保持未知。

## 一、事实（真实生产调用观测，非源码推导）

时间均为日志本地时间（UTC+8）；UTC = 本地 −8h。

| 观测 | 值 | 出处 |
| --- | --- | --- |
| 输入文件 | `asr_example_zh.wav`，下载成功 0.17 MB | `app.log` `downloaders.generic:download_file:717` |
| 下载器 | `downloaders.generic`（直链 wav） | 同上 |
| 时限传递 | `transcription_deadline duration=unknown fallback=sdk_auto` | `capswriter_client:_transcription_deadline:141`，两个失败任务各一条 |
| 终态消息 | `code=timeout, 原因: 转录超过自动预算：远端转录阶段超时` | `capswriter_client:log:648` |
| 尝试次数 | `尝试 1/5`，无重试 | 同上 |
| 转录阶段耗时 | `120190ms` / `120183ms` | `perf_tracker:track:68` |
| 任务时间 | `task_cc2eeeeb…` 00:22:18→00:24:19；`task_c3b6e0be…` 00:47:18→00:49:19 | 同上 |
| 生产 SDK 与 pin 一致 | 容器 `capswriter_asr/client.py` sha256 `ff476ad7cd40401b7ed77c7606cb4fc14dc3d529043c29c4714b199c13423cdf` == 本地 pin `858c6b9` 同文件 sha256 | `docker exec sha256sum` vs `sha256sum ~/.cache/uv/git-v0/…/858c6b9/…` |
| ASR 端点与探活 | `ws://<ASR_HOST>:6016`（内网端点，公开仓脱敏），GET `/health` → `protocol_version=2, role=server, encodings=[f32le,s16le,flac,ogg_opus], model=paraformer` | 生产 config 单键 grep + n305 单 URL GET |
| 对照组（自动预算下成功） | `task_9ec7a075…` m4a 走 sdk_auto，转录 321208ms 成功 | `perf_tracker` |

### 关键推论（有观测支撑，非猜测）

自动预算是 `max(120, duration + 60)`，`duration` 是**SDK 解码后的 PCM 样本数 / 16000**，不是文件容器时长（`client.py:418`）。所以：

- 5.55s wav → 预算 120 秒 → 实测 121 秒终态。**吻合**。
- 同期的 321 秒 m4a 成功、17621 秒 mp4 超时，也都落在 `duration + 60` 之下。**不构成反例**。

这解释了「为什么只有这个 wav 卡在 121 秒」：只有它短到撞上 120 秒下限。

## 二、本地无网络探针（真实调用，非手造预期）

环境：worktree 内 `uv sync --offline` 建的 `.venv`，Python 3.11，SDK pin `858c6b9`；ffmpeg 造 5.55s/16kHz/mono 正弦 wav（sha256 `c08fd13b…6aca7`，177678 bytes，ffprobe `5.550000`）。**未连任何真实 ASR，未碰生产。**

1. producer 侧（`/tmp/caps-short-probe-261004/probe_producer.py`，patch 的是 `transcribe_file_sync` 捕获实参）：
   - `media_duration=None` → kwargs 只有 `{encoding: flac, seg_duration: 25, seg_overlap: 2}`，**无 `deadline_total`**
   - `media_duration=0.0` → `deadline_total=120.0`（与自动预算不可区分的第二个候选，已排除：生产日志是 `duration=unknown` 而非 `duration=0.00`）
   - `media_duration=5.55` → `deadline_total=142.2`；`93.08898` → `492.35592`（与既有单测公式一致）
   - 真实执行 `GenericDownloader.download_file`（只 mock SSRF 之后的 `_safe_request`）：返回路径正常，`last_media_duration is None`，且 `inspect.getsource(GenericDownloader.download_file)` 不含 `_validate_media_file`
2. SDK 侧（`/tmp/caps-short-probe-261004/probe_sdk_deadline.py`：同端口假服务端，`/health` 返协议 v2，WS 收下音频帧后**永不回应**）：
   - 不传 `deadline_total` → **120.2s** 失败，`AsrError(code=timeout, 「转录超过自动预算：远端转录阶段超时」, retryable=False)`
   - 传 `deadline_total=142.2` → **142.2s** 失败，消息变为「转录超过deadline_total：…」
   - 服务端实收帧：单帧 `is_final=true, samples_total=88800, source=file, encoding=flac, seg_duration=25, seg_overlap=2`

即：生产那条日志的**文本与耗时**在本地被逐字复现，链路到「客户端主动放弃」为止闭合。

## 三、最靠近失败的已证环节与仍缺的证据

**已证环节**：`GenericDownloader` 不探测时长 → `media_duration=None` → 不传 `deadline_total` → SDK 自动预算 120 秒 → 客户端 timeout 显式失败。

**未知（不可由现有证据判定）**：服务端 `<ASR_HOST>:6016` 对这份 5.55s wav 的真实行为——是没收到、是收到了不处理、还是处理了但不发 final。`/health` 只证明进程活着，不证明该任务的服务端终态。

已被现有证据**排除**的说法：
- 「FunASR 成功 = CapsWriter 已修」：FunASR 是另一条链路，不构成 CapsWriter 的对照成功。
- 「121 秒说明时限太短」：短音频自动预算下限恒为 120 秒，与媒体长短无关；把它调大只会换一个数字，不能解释服务端为何不回。
- 「5.55s < file_seg_duration 25 是分段缺陷」：SDK 发的是**单帧** `is_final=true`、整段 flac 都在一帧里（`b64_bytes=56540`），分段由服务端做；本卡无任何证据指向分段。

**缺的最小实验（需主脑授权，本卡不做）**：向 `<ASR_HOST>:6016` 提交同一份 5.55s wav（生产 ASR 提交音频，本卡明令禁止），同一时刻抓服务端日志/结果，判定「服务端是否返回、何时返回」。若服务端其实在数分钟内正常返回，则根因转为客户端预算策略；若服务端根本不回，根因在服务端，本仓改动无效。

**输入样本可复核程度：低。** 生产日志只留下文件名 `asr_example_zh.wav` 与 0.17 MB 两项；该文件已随任务结束从 `data/temp` 清理（`ls data/temp` 为空），5.55s 来自验收输入资料，日志与任务表都没有媒体时长字段。本地探针用的是自造正弦 wav，只能复现**预算行为**，不能替代原样本。

## 四、修复归属建议（本卡不改代码）

- 归属**本仓**：`GenericDownloader` 全覆盖 `download_file` 而不 ffprobe，导致「拿不到时长 → 回落 120 秒下限」在直链短音频上必然发生。若确认服务端会正常返回，这就是最小修复点（在下载成功后对已落地文件做一次时长探测，或按已落地文件补探测）。
- 归属**服务端**：若上述实验显示服务端对短音频不返回，则本仓任何时限改动都只是把失败推后。
- 归属 **SDK**：暂无证据。SDK 行为已在本地与生产两侧对齐（sha256 一致 + 消息逐字复现）。
- **不提出**：通用兼容层、fallback 引擎、重试/超时堆叠——锁定决策已排除，且无第二个消费者。

## 四之二、同期 CapsWriter 任务时间线（同一份 app.log，只读投影）

| 本地时间 | 输入 | 时限传递 | 终态 | 转录阶段耗时 |
| --- | --- | --- | --- | --- |
| 10-03 19:31→19:42 | `tmpdz9b9mka.m4a` | `duration=unknown` → sdk_auto | 成功，生成 txt + funasr.json | — |
| 10-03 23:11→23:17 | `tmpr_ap5fu7.m4a` | `duration=unknown` → sdk_auto | 成功 | — |
| 10-04 00:22:18→00:24:19 | `asr_example_zh.wav` | `duration=unknown` → sdk_auto | **failed timeout** | 120190ms |
| 10-04 00:25:46→00:31:33 | `tmp4o_rfopr.m4a` | （窗口内） | 成功 | — |
| 10-04 00:31:19→05:25:00 | `今天超绝JK！.mp4`（494 MB） | `duration=unknown` → sdk_auto | **failed timeout** | 17621038ms |
| 10-04 00:31:24→00:36:45 | `tmpo4skv5c4.m4a` | `duration=unknown` → sdk_auto | 成功 | 321208ms |
| 10-04 01:23:17→01:28:51 | `tmpu24g4oow.m4a` | （窗口内） | 成功 | — |
| 10-03 18:14:58 | （前一日，`duration=93.09 value=492.4`） | 显式 deadline_total | — | — |

读法：唯一显式带上时长的是 93.09s 那次（#155 路径）；其余全部落回 sdk_auto。**失败的 wav 与成功的 m4a 走的是同一条代码路径、同一个 120 秒下限**，差别只在解码后音频长短——所以「短音频」在本例里的真实含义是「唯一撞上 120 秒下限的那个」。

## 四之三、上游与本仓的只读复核（未改任何一侧）

| 来源 | 结论 |
| --- | --- |
| 上游 issue #52（OPEN）「SDK flac/ogg 转码未限定音轨…长文件必然 120s 超时」 | 对应修法 PR #53 已合入 pin `858c6b9`；本例输入是纯 wav、无视频轨，不适用 |
| 上游 issue #43 / #55 / #56 / #61（OPEN） | 分别是容器时长估算、测试环境红、跨 ffmpeg 版本验证、HTTP 计费边界；均不涉及短音频客户端下限 |
| 上游 open PR #61~#64 | 全部在 HTTP 容量三闸、QA、baseline 采集线上，无「短音频时限」条目 |
| 本仓主干 `ff92a175` | 无相关提交；`tests/unit/test_capswriter_deadline_budget.py` 明确锁死「拿不到时长就不传 `deadline_total`」，即当前行为是**有意决策**而非疏漏 |
| 本仓 `GenericDownloader.download_file` | 全覆盖基类下载流程，源码内无 `_validate_media_file` 调用（源码检查 + 真实执行双证） |

## 五、锁定决策与已否决方案（持久记录）

锁定（本卡遵守）：
- 仅诊断，不实现修复；保留根因未知；不得把 FunASR 成功当 CapsWriter 修复；不得从 121 秒直接认定时限过短；不得从 5.55s < `file_seg_duration=25` 推定分段缺陷。

已否决（附理由）：
- 「调大自动预算 / 传更大的 `deadline_total`」：只改失败时刻，不改服务端行为；且现有测试 `test_unknown_duration_omits_deadline_kwarg` 明确把「拿不到时长就不传」锁成有意决策。
- 「换 FunASR 引擎绕过」：锁定决策排除，且属掩盖失败。
- 「重开 #155 套旧修法」：#155 修的是长媒体 3.29 倍实时的预算公式，其前提是拿得到时长；本例根本拿不到时长，前提不成立。
- 「上游已有同因修复，直接升级」：上游仓最近 PR（#52/#53/#59/#60 与 open 的 #61~#64）都在 HTTP 容量/QA/baseline 线上，无一条针对「短音频客户端 120 秒下限」；本仓主干亦无相关提交。

## 六、命令与退出码（可复核）

生产只读（n305，单命令单 URL，各自 `-o` 语义已在脚本内）：

| 命令 | 退出码 | 用途 |
| --- | --- | --- |
| `ssh n305 'docker ps --format … | grep -i video; ls data'` | 0 | 容器 running/healthy、日志目录 |
| `ssh n305 'grep -n "\"server_url\"" config/config.jsonc'` | 0 | ASR 端点（18 行= capswriter 6016，165 行= funasr 8767） |
| `ssh n305 'grep -c "task_cc2eeeeb…" logs/app.log'` | 0 | 30 行命中 |
| `ssh n305 'grep -E "开始转录文件|转录完成，生成文件|转录文件失败" logs/app.log'` | 0 | 转录时间线白名单 |
| `ssh n305 'grep -c "transcription_deadline" logs/app.log'` | 0 | 9 行，8 行 `duration=unknown` |
| `ssh n305 'curl -s --max-time 8 http://<ASR_HOST>:6016/health | python3 -c …'` | 0 | 只投影 4 个白名单字段 |
| `ssh n305 'docker exec … sha256sum …/capswriter_asr/client.py'` | 0 | 与 pin 比对 |

本地（worktree）：

| 命令 | 退出码 | 用途 |
| --- | --- | --- |
| `uv sync --offline` | 0 | 建 `.venv` |
| `PYTHONPATH=src uv run --no-sync python /tmp/caps-short-probe-261004/probe_producer.py` | 0 | producer 实参 + GenericDownloader 探测缺失 |
| `PYTHONPATH=src uv run --no-sync python /tmp/caps-short-probe-261004/probe_sdk_deadline.py` | 0 | 120.2s/142.2s timeout 复现 + 实收帧 |

上游只读：`gh issue view 166`、`gh api repos/zlxlabs/CapsWriter-ASR-Server/commits|pr list|issue list`（仅读，未评论未改）。

## 七、继承红与新红

卡面写明主干基线 API 不可用（`gh api request failed`），因此**继承红未能判定**。本卡无代码改动、未跑全量测试（`make test` 未执行），因此不存在新红判定依据。

## 八、踩到的坑

1. 卡面要求先读的两个技能文件 `~/.pi/agent/skills/{deploy-ops,investigate}/SKILL.md` 均 `Permission denied`，无法读取；按主脑全局约定自行执行同口径纪律。
2. 探针初版用 `aiohttp` 起假服务端，dev 依赖里没有（aiohttp 在 perf extra）→ `ModuleNotFoundError`；改用已装的 `websockets 15` 的 `process_request` 钩子同端口兼做 `/health`。
3. `transcribe_file_sync` 内部是 `asyncio.run()`，在已有 loop 里调用直接 `RuntimeError: asyncio.run() cannot be called from a running event loop`，且**报错同时把 coroutine 泄漏成 never-awaited 警告**——若只看 warning 会误以为超时逻辑有问题。改用等价 async 入口 `transcribe_file`。
4. `websockets.http11.Response` 的 headers 必须传 `Headers` 对象，传 `dict` 会在握手时抛 `AttributeError: 'dict' object has no attribute 'serialize'`。
5. 首版 mock 了 `requests.get`，但 `GenericDownloader` 走 `_safe_request`/`_dispatch_pinned_request` 钉 IP 路径，结果**真的解析 DNS 并向 example.com 发了 3 次请求**（全部 404）。已改为 mock `_safe_request`；这三次外网请求是本卡唯一的意外出网，无凭据、无生产数据。
6. `GenericDownloader.__init__` 需要 `config/config.jsonc`（本 worktree 没有，且本卡不改配置），改用 `__new__` 绕过并手工补基类初值。

## 九、闸与绕过

- 只写两个文件：本报告与 `progress/caps-short-progress.md`。未改任何业务代码、依赖、config、CI。
- 生产侧只有：容器/日志只读 grep、固定 task id 过滤、单个 `/health` GET、容器内文件 sha256。未 POST 新任务、未向 ASR 提交音频、未重启/部署、未操作 issue、未读取 users.json 或任何凭据（`server_url` 为主机名+端口，非凭据）。
- 日志只投影白名单字段（时间、级别、模块行号、文件名、错误码、耗时），task id 与文件路径做前缀截断，未回显原始日志全文。
- 预算：定向检索约 20 次、SSH 8 次（限 12），均在卡面预算内；墙钟约 50 分钟，**超出 40 分钟软限**，缺口在于两次 SDK 探针重写与 262 秒的等待。

## 十、与卡面的偏差

- 卡面要求「比较历史任务运行版本」：历史任务运行的就是 `ff92a175` = 当前 HEAD，无版本差；改为核对**容器内实际安装的 SDK 文件 sha256** 与 pin 一致（更强的一层）。
- 卡面禁止向生产 ASR 提交音频，故缺的那一环实验只能写成方案交主脑裁决，未执行。
- 输入样本复核降级为「低」：原 wav 已清理，无法取 hash。

## 十一、最贵的一步

为把「121 秒」从数字变成机制，串起了四份互相独立的证据：生产 `_transcription_deadline` 与 `code=timeout` 两条日志、容器 SDK 与 pin 的 sha256 一致性、本地假服务端把 120.2 秒与逐字错误消息复现出来、以及用同期 321 秒成功 / 17621 秒超时两个任务反证「自动预算是 `max(120, 解码时长+60)`、只有短音频撞下限」。任何一环缺失都只能停在「疑似时限」。
