# #166 CapsWriter 短音频诊断 进度

- 派发：`dlg-20261004-023618-c714a0`；基线 `ff92a175`（= 生产运行版本）；分支 `card/caps-short-diagnose-261004`。

## 阶段一：现场与证据链（已完成）

- 结论：`121 秒` 已被解释到客户端计时边界——直链 wav 走 `GenericDownloader`，无 ffprobe → `media_duration=None` → 不传 `deadline_total` → SDK 自动预算下限 `max(120, 解码时长+60)=120` → `AsrError(timeout, 转录超过自动预算：远端转录阶段超时)`，`retryable=false` 故只试一次。
- 生产观测：`_transcription_deadline duration=unknown fallback=sdk_auto`、`code=timeout` 原文、`尝试 1/5`、perf `120190ms`/`120183ms`；容器 SDK `client.py` sha256 与 pin `858c6b9` 逐字一致。
- 本地无网络探针：producer 实参（`None`→无 `deadline_total`；`0.0`→`120.0`；`5.55`→`142.2`）、`GenericDownloader.download_file` 不探测（`last_media_duration is None`）、假服务端复现 120.2s/142.2s 超时与逐字消息、实收帧 `is_final=true, samples_total=88800`。

## 阶段二：定性与收尾（已完成）

- 决策：根因保持**未知**——服务端对这份 5.55s wav 为何不给 final 未证；本卡禁止向生产 ASR 提交音频，该实验写成方案交主脑裁决。
- 已否决：调大预算/重试堆叠、换 FunASR 绕过、套 #155 旧修法（其前提是拿得到时长）、「上游已有同因修复」（上游在办 PR 均为 HTTP 容量/QA 线上）。
- 归属建议：本仓（`GenericDownloader` 不探测）**或**服务端，取决于缺失实验的结果；SDK 无证据。
- 输入样本可复核程度=低：原 wav 已随任务清理，日志只留文件名与 0.17 MB。

## 下一步（需主脑裁决）

1. 是否授权一次生产 ASR 提交实验（同 wav + 服务端侧观测）以判定服务端终态。
2. 若确认服务端会正常返回 → 本仓补最小时长探测（不引入兼容层/fallback）；若服务端不回 → 转上游/服务端侧，本仓不改。

## 过程记录

- 卡面要求读的两个技能文件权限拒绝（`Permission denied`），按全局纪律自行执行。
- 探针三次返工：aiohttp 不在 dev 依赖 → `asyncio.run` 不能嵌套 loop → `websockets.Response` headers 需 `Headers` 对象。
- 一次 mock 位置错误导致真的向 example.com 发了 3 次请求（已改 mock `_safe_request`），无凭据无生产数据。
- 继承红：卡面主干基线不可用，**未能判定**；本卡无代码改动、未跑全量测试。
