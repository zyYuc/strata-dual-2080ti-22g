# 0.1.44.1 · 128K 筛选轮 1（2026-10-14 夜）

> 背景：用户指出 60K 时 prefill 仍在爬升（2K 720 → 60K 2131 单调递增），60K 不足以判别；
> 改为**只用 128K 筛选**。以下均为 128K（target_words=128000，实测 prompt_tokens≈132992）× 2 runs，
> max_tokens 1024，含 20K 暖机，引擎 0.1.44.1（296b59d2，sm_75/CUDA 13.0 源码编译）。

## 结果（128K prefill tok/s，两 run）

| 条件 | run1 | run2 | mean | vs base | TTFT (s) |
|---|---|---|---|---|---|
| **base（0.1.44.1 终配）** | 2226 | 2160 | **2193** | — | 59.8 / 61.5 |
| `STRATA_PREFILL_LEND_PCT=95` | 2207 | 2143 | 2175 | −0.8% | 60.3 / 62.0 |
| `STRATA_PREFILL_LEND_PCT=100` | 2194 | 2140 | 2167 | −1.2% | 60.7 / 62.1 |
| `STRATA_SPLIT_RING=512` | 2179 | 2122 | 2151 | −1.9% | 61.1 / 62.6 |
| `STRATA_SPLIT_RING=768` | 2181 | 2125 | 2153 | −1.8% | 61.0 / 62.5 |

结论：ring 与 lend_pct 在 128K 上**均无增益**，base 最佳。run 间噪声约 ±66 tok/s（±3%），
因此上述负值都在噪声边缘，但没有任何一项跑赢 base，不值得采纳。

## 附带确认

- 128K base = 2193 略高于 0.1.42 记录的 2163（+1.4%）。
- 0.1.44.1 五档基线（同夜测得）：2K 720 / 5K 1057 / 8K 1118 / 20K 1708 / 60K 2131 / 128K 2193。

## 未完成项（被硬件事故中断）

第 2 轮计划测 `STRATA_PREFILL_AUTO_MAX=16384 / 32768`、`PREFILL_STREAM_AHEAD=0`、
`PREFILL_GROUP_GATHER=0`。**第一项 `AUTO_MAX=16384` 在 128K 下把台机打到内存 thrash、
sshd 失联（系统假死）**，与 lessons.md 既有结论一致（48GB 机器上 16384 chunk 会 thrash 借槽区域，
需要 92GB 级全锁页机器）。此项应**永久否决**，不再重复测试。

## 复现命令

```bash
python3 /tmp/run_context_ttft_fair.py --base-url http://127.0.0.1:8000 --model qwen-local \
  --word-counts 128000 --runs 2 --max-tokens 1024 --output /tmp/k128-<tag>.json
```
（env 通过 config `strata-swift-iq3_xxs.json` 的 `env` 块注入，改后需重启 strata.service；
基础 env：SPLIT_OWN=0 / DEBUG=1 / STAGE_TRIM=1 / PREFILL_HELP=1 / SELECT_SIMT=1 / SPLIT_MTP_BATCH=1 / SPLIT_RING=384）
