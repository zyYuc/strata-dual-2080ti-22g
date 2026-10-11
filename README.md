# Strata 双 RTX 2080 Ti 22G 调优配置 · Qwen3.8-Flash-Next (Swift IQ3_XXS)

> 双卡层切分 + 262K 上下文 + MTP 投机解码，**60K prefill 2116 tok/s / 128K 2163 tok/s / decode ~77-84 tok/s**（48 GB 内存消费级平台）。
> 引擎 Strata v0.1.42，全部数字为本机实测（`bench/` 有全部原始 JSON），口径见下。

## 硬件

| 项 | 配置 |
|---|---|
| GPU | 2× RTX 2080 Ti 22G 魔改（Turing sm_75，200W/卡，NVLink 桥在位但 layer split 不使用） |
| CPU | Ryzen 7 5700X（Zen3，8C16T，AVX2） |
| 主板 | X570，双卡 PCIe Gen3 x8+x8（实测 6.7 GB/s） |
| 内存 | 48 GB DDR4-3600（DOCP，4 DIMM 混插） |
| 系统 | Ubuntu 26.04.1，驱动 580.178.04，CUDA 13.0，gcc 15.2 |

## 为什么快（按贡献排序）

1. **v0.1.42 双卡借槽 RAM 备份**：`--resident-experts` 下引擎自动把 prompt path 借走的专家槽全部在锁页内存留副本（启动日志 `lend sizing: borrows 4934 slots, 4934 keep their experts in RAM too`），24576 个专家 100% 被显存+锁页覆盖，推理期零专家文件读。零参数，升级即得。
2. **`STRATA_SELECT_SIMT=1`**：QSA FP32 tiled 评分器（0.1.40.3 起 opt-in，官方 OLDER_GPUS.md 有 Turing 实测）。本机 60K +11%、128K +20%、decode +1.7%。**这是最容易漏的开关**。
3. **`STRATA_SPLIT_MTP_BATCH=1`**：双卡 MTP 批处理，prompt +2~5%（5K-60K），decode 平。
4. **`STRATA_SPLIT_RING=384`**：借槽环 96→384，20K +3.5%、128K +2.1%。
5. **`--aux-cpus auto`**（0.1.42 新增）：decode +3.6%，prefill -1~4%。
6. 基础项：`STRATA_BF16_TC=1`（Turing 必开，prompt +20~34%）、`--pipeline-windows 2`、split 26、`--pcie-frac 0.35`、`--spec-min-p 0.70`（均经 calibrate/实测确认）。

## 环境变量（全量）

```json
"env": {
  "STRATA_BF16_TC": "1",          // 或 systemd drop-in
  "STRATA_SELECT_SIMT": "1",
  "STRATA_SPLIT_MTP_BATCH": "1",
  "STRATA_SPLIT_RING": "384",
  "STRATA_SPLIT_OWN": "0",        // 显式借用；不设会 auto 切 own 改变驻留行为
  "STRATA_STAGE_TRIM": "1",       // 0.1.42 上 2K prefill 承重墙（删则 -19%）
  "STRATA_PREFILL_HELP": "1",     // CPU 协助 1-3.3K prompt MMQ（同上）
  "STRATA_DEBUG": "1"             // 可选，调试日志
}
```

注：`STRATA_ROUTE_TAIL_SKIP` 不设 = 0.1.42 默认开（本机 77% 驻留下中性；会轻微改变输出，逐字复现旧版设 0）。

## 关键加载参数

```
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.70 \
--pipeline-windows 2 --max-context 262144 --kv int8 --resident-experts \
--vram-reserve-mib 700 --pcie-frac 0.35 --ple-row-cache 8388608 --ple-inflight 1024 \
--conversation-cache-mib 13312 --conversation-cache-slots 4 --aux-cpus auto \
--layer-split 26 --vision
```

完整 config 见 [`config/strata-swift-iq3_xxs.example.json`](config/strata-swift-iq3_xxs.example.json)。

**注意 `--prefill auto` 保持 8192**：`auto:16384` 在 48 GB 内存上三个引擎版本实测 20K prefill -19~23%（92 GB 全锁页机器才扛得住）；`--kv-resident 32768` 同理（decode -6~9%，decode 受主机侧专家流带宽约束）。

## 实测结果（五档 ×2 + 128K 单发）

| 档位 | prompt tokens | prefill tok/s（均值） | decode tok/s | TTFT |
|---|---|---|---|---|
| 2K | 2124 | 665 | ~80 | 2.9-3.5 s |
| 5K | 5216 | 1010 | 80.3 | 5.1-5.2 s |
| 8K | 8355 | 1095 | 83.4 | 7.6-7.7 s |
| 20K | 20849 | 1706 | 75.7 | 12.2-12.3 s |
| 60K | 62420 | **2116** | 76.5 | 29.3-29.7 s |
| 128K | 133080 | **2163** | 70.7 | 61.5 s |

对 0.1.41 原配置的累计提升：5K +6.7% / 8K +11.6% / 20K +14.5% / 60K +16.3% / 128K +22.8%，decode 持平略升。

## 测速口径

`bench/run_context_ttft_fair.py`（本仓库附带）：流式、`prefill = prompt_tokens / TTFT`（TTFT=首个思考字符）、`decode = completion_tokens / (total − TTFT)`、max_tokens 1024、每 run 换 seed 防前缀缓存。五档 = 2000/5000/8000/20000/60000 words × 2 runs + 20K 暖机。

```bash
python3 bench/run_context_ttft_fair.py --base-url http://<host>:8000 \
  --model qwen-local --word-counts 2000 5000 8000 20000 60000 \
  --runs 2 --max-tokens 1024 --output bench/results/your-run.json
```

## 已测否决（别重复踩）

| 候选 | 结果 |
|---|---|
| `--prefill auto:16384` | 20K prefill -19~23%（0.1.40/0.1.41/0.1.42 三版本一致，92 GB 机器专属） |
| `--kv-resident 32768` | decode -6~9%（48 GB DDR4 扛不住 KV 流式；92 GB DDR5 机器在用） |
| `--pool-workers 5`（calibrate 推荐） | prefill -7%，decode 噪声内 |
| calibrate 写回的其余组合 | 短 prompt 口径调优，本机全档变差，须本机复测裁决 |
| `STRATA_ROUTE_RESIDENT/PRIOR=0.5` | 满覆盖栈上中性，且改输出（KL ~0.0085） |

## 复现要点

1. 引擎 ≥ v0.1.42（借槽 RAM 备份是 0.1.42 双卡新特性），源码编译 `sm_75`。
2. 启动日志验收：`FileExpertSource: the split stages lend N slots: N keep their experts in RAM too`（M=N 满覆盖）、`mapped pinned cache complement ready: resident 16-17 GiB`、`layer split: 77% of the experts resident`。
3. 显存两卡应 ~96% 占满（auto 每卡留 700 MiB），内存补集 16-17 GiB 锁页。
4. 200W 功耗墙（改装卡散热优先）——250W 曾测得 +3~5%，自担风险。
