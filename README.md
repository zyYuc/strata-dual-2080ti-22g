# Strata 双 RTX 2080 Ti 22G 调优配置 · Qwen3.8-Flash-Next (Swift IQ3_XXS)

> 双卡层切分 + 262K 上下文 + MTP 投机解码，**60K prefill 2131 tok/s / 128K 2193 tok/s / decode ~76-84 tok/s**（48 GB 内存消费级平台）。
> 引擎 Strata **v0.1.44.1**，全部数字为本机实测（`bench/` 有全部原始 JSON），口径见下。

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
  "STRATA_STAGE_TRIM": "1",       // 2K prefill 承重墙（删则 -19%）
  "STRATA_PREFILL_HELP": "1",     // CPU 协助 ≤3.3K prompt（超过 3.3K 自动失效，见下）
  "STRATA_DEBUG": "1"             // 可选，调试日志
}
```

注：`STRATA_ROUTE_TAIL_SKIP` 不设 = 0.1.42 起默认开（本机 77% 驻留下中性；会轻微改变输出，逐字复现旧版设 0）。

**`STRATA_PREFILL_HELP` 的真实作用域（源码核实）**：`prefill.cpp` 的 `split_help_frac()` 规则是 `min(0.45, 0.5 - T/16384)`，低于 0.3 时返回 0 —— 即 **chunk ≥ 3277 token 时该开关自动失效**。本机 `--prefill auto` 用 8192 chunk，所以它只在 ≤3.3K 的短 prompt 上工作（这正好解释了它 2K 档 +36% 而长档持平/略负的实测）。若想在长档强制启用需设 `STRATA_PREFILL_HELP_FRAC=<0..1>`（本仓库未测，留作待验证项）。

## 关键加载参数

```
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.70 \
--pipeline-windows 2 --max-context 262144 --kv int8 --resident-experts \
--vram-reserve-mib 700 --pcie-frac 0.35 --ple-row-cache 8388608 --ple-inflight 1024 \
--conversation-cache-mib 13312 --conversation-cache-slots 4 --aux-cpus auto \
--layer-split 26 --vision
```

完整 config 见 [`config/strata-swift-iq3_xxs.example.json`](config/strata-swift-iq3_xxs.example.json)。

**注意 `--prefill auto` 必须保持 8192**：`auto:16384` / `auto:32768` 在 48 GB 内存上会把机器打到内存 thrash（详见下方"内存安全红线"）；`--kv-resident 32768` 同理（decode -6~9%，decode 受主机侧专家流带宽约束）。

## 实测结果（v0.1.44.1，五档 ×2 + 128K）

| 档位 | prompt tokens | prefill tok/s（两 run） | 均值 | decode tok/s | TTFT |
|---|---|---|---|---|---|
| 2K | 2122 | 672 / 768 | 720 | 78.3 | 3.16 / 2.76 s |
| 5K | 5222 | 1056 / 1058 | 1057 | 79.0 | 4.94 / 4.94 s |
| 8K | 8340 | 1122 / 1113 | 1118 | 81.7 | 7.45 / 7.48 s |
| 20K | 20846 | 1705 / 1710 | 1708 | 78.4 | 12.23 / 12.19 s |
| 60K | 62388 | 2160 / 2102 | **2131** | 72.6 | 28.90 / 29.66 s |
| 128K | 132992 | 2226 / 2160 | **2193** | 73.5 | 59.78 / 61.54 s |

原始数据：[`bench/results/01441/strata-01441-baseline-5tier.json`](bench/results/01441/strata-01441-baseline-5tier.json)、128K 见 [`bench/results/01441/`](bench/results/01441/)。

## 版本演进（0.1.42 → 0.1.44.1）

同一硬件、同一 config、纯升级引擎（源码编译 sm_75/CUDA 13.0）：

| 档位 | 0.1.42 | 0.1.44.1 | 变化 |
|---|---|---|---|
| 2K | 665 | 720 | **+8.3%** |
| 5K | 1010 | 1057 | **+4.7%** |
| 8K | 1095 | 1118 | **+2.0%** |
| 20K | 1699 | 1708 | +0.5% |
| 60K | 2114 | 2131 | +0.8% |
| 128K | 2163 | 2193 | +1.4% |

短 prompt 受益最明显（0.1.43.1 修了 prompt 路径的若干调度问题），长档基本持平。
**升级注意**：0.1.44 把 setup 的默认量化从 IQ3_XXS 换成了 IQ3_S（慢 10-15%、多吃约 7 GB RAM），
0.1.44.1 才修复"重跑 setup 保持已装尺寸"。手写 config 的用户升级时务必核对模型文件未被替换。

## 测速口径

`bench/run_context_ttft_fair.py`（本仓库附带）：流式、`prefill = prompt_tokens / TTFT`（TTFT=首个思考字符）、`decode = completion_tokens / (total − TTFT)`、max_tokens 1024、每 run 换 seed 防前缀缓存。五档 = 2000/5000/8000/20000/60000 words × 2 runs + 20K 暖机。

```bash
python3 bench/run_context_ttft_fair.py --base-url http://<host>:8000 \
  --model qwen-local --word-counts 2000 5000 8000 20000 60000 \
  --runs 2 --max-tokens 1024 --output bench/results/your-run.json
```

**筛选建议用 128K 单档**：prefill 速率随 context 单调爬升（2K 720 → 60K 2131 → 128K 2193），
60K 时仍在上升段、区分度不足；128K 才是判别档。run 间噪声约 ±3%，小于 3% 的差异需多轮才能判定。

## 已测否决（别重复踩）

### 内存安全红线（重要）

| 候选 | 结果 |
|---|---|
| `STRATA_PREFILL_AUTO_MAX=16384` / `=32768` | **128K prompt 下把 48 GB 机器打到内存 thrash、sshd 失联、需硬重启**（两次实测）。48 GB 平台绝对不要碰：大 chunk 的借槽需求会把 pinned 内存 + 借槽区域一起撑爆。92 GB 级全锁页机器专属。 |

引擎每轮启动会分配 ~17.4 GiB **page-locked** 内存（`free` 里显示为 `shared`，不计入 `used`），
128K 请求再叠加 prompt buffer + 4934 借槽。**监控内存时不能只看 `available`，必须同时看 `shared` 与 `MemAvailable`**；
`shared` 持续接近总内存即处于危险区。

### 128K 实测无增益（v0.1.44.1，mean of 2）

| 候选 | 128K prefill | vs base 2193 |
|---|---|---|
| `STRATA_PREFILL_LEND_PCT=95` | 2175 | −0.8% |
| `STRATA_PREFILL_LEND_PCT=100` | 2167 | −1.2% |
| `STRATA_SPLIT_RING=512` | 2151 | −1.9% |
| `STRATA_SPLIT_RING=768` | 2153 | −1.8% |
| `STRATA_PREFILL_STREAM_AHEAD=0` | 2186 | −0.3% |
| `STRATA_PREFILL_GROUP_GATHER=0` | 2171 | −1.0% |
| `STRATA_PREFILL_ISSUER=0` | 2166 | −1.2% |

**本机 77% 驻留 + 满覆盖（零文件读）的栈上，这 7 个调度类开关全部无增益或略负** —— 现有配置已接近该引擎版本在此硬件上的稳态上限。

### 早期否决（0.1.42 及以前）

| 候选 | 结果 |
|---|---|
| `--prefill auto:16384` | 20K prefill -19~23%（0.1.40/0.1.41/0.1.42 三版本一致，92 GB 机器专属） |
| `--kv-resident 32768` | decode -6~9%（48 GB DDR4 扛不住 KV 流式；92 GB DDR5 机器在用） |
| `--pool-workers 5`（calibrate 推荐） | prefill -7%，decode 噪声内 |
| calibrate 写回的其余组合 | 短 prompt 口径调优，本机全档变差，须本机复测裁决 |
| `STRATA_ROUTE_RESIDENT/PRIOR=0.5` | 满覆盖栈上中性，且改输出（KL ~0.0085） |
| `--peer-device` | ≤20K 最快但 60K 退 20%（NV2 半速 NVLink），本机不上线 |

## 待验证项

- `STRATA_PREFILL_HELP_FRAC=0.2~0.3`：强制让 split-help 在 8192 chunk 上工作（默认在 ≥3277 token 时关闭）。有理论依据，但未实测。
- `STRATA_MMQ_RESIDENT_SORT_NE=1`：要求"该层专家全部驻留"，本机 77% 驻留不满足前置条件。
- 96 GB 内存路径：60K/128K 的残余瓶颈是内存容量（page cache 与专家工作集无余量），未验证。

## 复现要点

1. 引擎 ≥ v0.1.44.1，源码编译 `sm_75`（`-DCMAKE_CUDA_ARCHITECTURES=75`），vision 引擎需单独构建（`cmake -S tools/vision -B build-vision -DSTRATA_VISION_CUDA=ON -DSTRATA_PORTABLE=OFF`）。
2. 启动日志验收：`FileExpertSource: the split stages lend N slots: N keep their experts in RAM too`（M=N 满覆盖）、`mapped pinned cache complement ready: resident 16-17 GiB`、`layer split: 77% of the experts resident`。
3. 显存两卡应 ~96% 占满（auto 每卡留 700 MiB），内存补集 16-17 GiB 锁页。
4. 200W 功耗墙（改装卡散热优先）——250W 曾测得 +3~5%，自担风险。
5. **测试纪律**：每个 A/B 条件跑完后先查 `free -g`（重点看 `shared`）与服务存活，再开下一个；改 config 后**立即 restart**，不留带毒配置窗口。
