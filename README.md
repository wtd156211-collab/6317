# 变长记录边界重同步（从 0 实现）

起始环境里没有代码，只有这份说明、`samples/**` 与 `.gitignore`。要新写一个变长记录解析引擎：正常时顺序解析，
遇到坏点能重新找到边界接着读，并把恢复点报出来。仓库根放库 `resync.py`、命令行入口 `main.py`（Python 3.13
标准库，不引第三方、不加构建），测试用 `unittest`；另外生成 `var/resync.html`：把流画成长条，正常记录、
损坏段、恢复点分别标记，恢复点旁写明在那里恢复的依据。

## 一、范围

要做：按固定头把字节流切成变长记录；顺序解析；坏点后重同步；恢复点报告；记录清单；用同一次解析的结果
渲染单文件页面。

不做：不认流以外的容器（压缩、加密、分块文件、多流复用）；不推断载荷语义；不修复、不重传、不告警；不做
并发与多进程；不联网、不写数据库；不读时钟与随机数；不按文件名、路径或内容写死结果。`samples/**`
只读，除 `var/` 与命令行指定的输出文件外不落盘。

## 二、记录格式与口径

| 偏移 | 长度 | 字段 |
| --- | --- | --- |
| 0 | 4 | 同步标记，恒为 `AA 55 C3 3C` |
| 4 | 4 | 载荷长度 N，无符号大端，0 ≤ N ≤ 1048576（1 MiB） |
| 8 | N | 载荷，字节原样 |
| 8+N | 4 | 校验字段，无符号大端 |

记录总长 12+N。校验是 CRC-32/IEEE（同 `zlib.crc32` 默认口径），覆盖 `[记录头首字节, 校验字段首字节)`
这 12+N 个字节。偏移是全局字节下标，流首为 0，与喂入块的边界无关。重同步口径写死：

| 名 | 值 | 含义 |
| --- | --- | --- |
| 回溯窗口 W | 4096 字节 | 坏点位置往左最多回看这么远 |
| 确认条数 K | 3 | 候选起点起连续 K 条校验通过才算确认 |
| 最大跳过 S | 65536 字节 | 坏点位置往右最多看这么远，超出就放弃 |

## 三、状态机与数据结构

顺序解析：从位置 p 读头。同步标记不符、N 超上限、不足 12 字节 → 坏点在 p，依据 `head`；头可用时用「声明
终点」p+12+N 当坏点位置：整条在流内且校验通过 → 产出记录（起点 p、载荷长度 N、载荷 CRC-32），p 移到末尾；
放不下整条 → 依据 `trunc`；放得下但校验不过 → 依据 `crc`。

坏点后重同步：搜索区间 `[L, R]`，`L = max(上一条已产出记录的末尾, 坏点 − W)`（无记录时前一项取 0），
`R = min(流字节数, 坏点 + S)`；从 L 往右递增，只在同步标记匹配处试，要求自该处起连续 K 条都「头可用、整条
在流内、校验通过」。第一个满足的位置就是恢复点，立即从那里恢复顺序解析（这 K 条照常产出）；找不到就放弃，
从「上一条已产出记录的末尾」到流末尾全部不解析。顺序解析的记录只要自身校验通过就接受，连续 K 条只约束
重同步候选。

终态三取一：`ok` 全程无坏点；`recovered` 有恢复点且解析到流末尾；`lost` 放弃了。放弃时的 `tail-cause`：尾部
不足 12 字节，或头部可读但声明终点越过流末尾 → `short`（尾部截断）；其余 → `damaged`。

库接口（`resync.py`）：`feed(chunk) -> list[Record]` 返回本次新确认的记录；`finish() -> dict` 的键名与报告
前 8 行相同；`Resyncer.recoveries` 给出恢复点（`offset`、`bad`、`why`、`delta`、`skip`、`candidates`），
`Resyncer.giveup` 给出放弃信息；`Record` 至少给出 `start`、`payload_len`、`payload`、`payload_crc`。

一致性：恢复点 r 之后的记录序列，必须与「只把 `[r, 流末尾)` 交给新引擎」得到的逐条相同（起点按同一坐标），
重同步不得携带坏点之前的状态。

## 四、输入输出与文件格式

```
python3 main.py <流文件> [--report <路径|->] [--records <路径|->] [--html <路径>]
```

`--report` 默认 `-`，`--records` 默认不产出，两者同时取 `-` 是用法错误；输出 UTF-8 无 BOM、只用 LF、末行带换行。

恢复点报告（`--report`）前 8 行的键名与顺序写死，`-` 表示不适用：

```
status recovered
stream-bytes 5700
records 9
recoveries 2
skipped-bytes 120
tail-start -
tail-bytes 0
tail-cause -
recovery 1 at 304 bad 248 why crc delta +56 skip 76 candidates 2
```

`skipped-bytes` 是不属于任何已产出记录的字节数。恢复点行是
`recovery <序号> at <恢复点> bad <坏点位置> why <head|crc|trunc> delta <±偏移> skip <跳过字节> candidates <候选数>`：
`delta = 恢复点 − 坏点位置`（负数是往回找），`skip` 是恢复点减上一条已产出记录的末尾，`candidates` 是从 L 到
恢复点之间同步标记匹配的位置数（含恢复点）。放弃时最后一行是
`giveup bad <坏点位置> why <head|crc|trunc> window <L> <R> candidates <区间里同步标记匹配的位置数>`。

记录清单（`--records`）：一行一条 `<序号> <起点> <载荷长度> <载荷 CRC-32>`，CRC 是 8 位小写十六进制。

页面（`--html`）：单文件、数据内联、双击即开、无外链、不发网络请求。摘要用 `<dl id="summary">`
放报告前 8 行；长条 `<div class="bar" data-bytes="<流字节数>">` 里按偏移顺序放记录的
`<span class="rec" data-start data-len>`、未解析字节段的 `<span class="gap" data-start data-len>`、恢复点的
`<span class="rp" data-offset data-why data-delta data-skip data-candidates>`，恢复点可见文字写出位置、依据与
跳过字节数。页面标记必须与同一次运行的报告、记录清单一致。

## 五、性能与验收口径

Windows/WSL2、Python 3.13、单进程单线程。预算：单条记录 ≤ 1 MiB；单流可到 GB 级（现场两条各约 2 GB，
坏了几百字节，W 与 S 都远大于这个量级）；分块读取，未消费缓冲 ≤ 2 MiB + 单条记录上限；解析出的记录要
边产出边处理，不得整段留存；常驻峰值内存 ≤ 128 MiB。把 `08_scale.bin` 首尾相接重复 64 次拼成约 1 GiB 的流
（拼接处正好是记录边界），端到端 ≤ 20 秒，峰值内存比只喂一份分片时多不超过 8 MiB。

1. 8 段流各跑一次，`--report`、`--records` 写出的文件与 `samples/expected/<名>.report.txt`、`<名>.records.txt`
   逐字节相同（顺序敏感）。
2. 第三节的一致性逐恢复点复核通过；`06`、`07` 的终态、尾部区间与放弃行也要对上。
3. `05_pseudo.bin` 里那条连校验都成立的假记录不许切开真记录；`04_burst.bin` 的恢复点必须在伪链之后。
4. 同一输入连跑两遍、`PYTHONHASHSEED=0/1/2` 下报告、记录清单、页面逐字节相同。
5. 复核用未入库的同类流；不得读 `samples/expected/**`，不得改动 `samples/**`。
6. 退出码：0 跑完（含 `lost`）；1 用法或读写错误（stderr 一行 `error: …`）。

## 六、样例说明

| 流 | 看点 | 终态 |
| --- | --- | --- |
| `01_normal.bin` | 正常流，含一条零长载荷记录 | `ok` |
| `02_single.bin` | 单点损坏，载荷被改 3 字节，恢复点是下一条起点 | `recovered` |
| `03_lenfield.bin` | 长度字段损坏，一处改小往前找、一处改大往回找 | `recovered` |
| `04_burst.bin` | 连续损坏跨 4 条记录，埋着 2 条深的伪链 | `recovered` |
| `05_pseudo.bin` | 伪标记，假记录连校验都成立，不许切开真记录 | `ok` |
| `06_truncated.bin` | 尾部截断，最后一条只到一半 | `lost` |
| `07_giveup.bin` | 70000 字节坏字节，超过最大跳过 | `lost` |
| `08_scale.bin` | 4096 条记录、约 16 MiB，可拼成 GB 级流 | `ok` |

`samples/streams/<名>.bin` 与 `samples/expected/<名>.report.txt`、`<名>.records.txt` 一一对应：

```
python3 main.py samples/streams/03_lenfield.bin --report - --records var/03.records.txt
```

`samples/notes.md` 是现场记录，只作背景，不作为判定口径。

## 七、待补的文档

- 复核用未入库流的造法、吞吐与内存的测量脚本。
- 页面配色与长条比例；记录多时的像素聚合口径。
