# kexports —— 官方内核/模块导出符号面（vendored 参考数据）

本目录存放**各平台官方内核与官方模块的导出符号表**，供发布门禁 `ainas-modules/scripts/gate_release.py`
判定"我们自编模块的未定义符号（UND）是否有提供者"。

## 为什么需要它

官方 DSM 内核的 `vmlinux` **全部是 stripped 的**（无 `.symtab`、无内嵌 IKCONFIG），符号名只能从
`__ksymtab` / `__ksymtab_gpl` 节提取。`official/hda1`、`official/rd` 只随附模块、**不随附 config 与内核本体**，
所以内核自身的导出面必须单独固化下来，否则门禁在 CI 里无法自足运行。

官方内核 `CONFIG_MODVERSIONS=n`（无 `__kcrctab`），因此**只需比对符号名，没有 CRC 维度**。

## 文件命名与来源

| 文件 | 内容 |
|---|---|
| `exports_<平台>_<两位版本>.txt` | 官方**内核**导出面（官方 `zImage` 解出的 vmlinux 的 `__ksymtab`） |
| `exports_mod_<平台>_<两位版本>.txt` | 官方**模块**导出面（`official/hda1` ∪ `official/rd` 的 `__ksymtab_<name>`） |
| `exports_cks_<平台>_<两位版本>.txt` | 我们自编 **cks 内核**导出面 —— 落 `ainas-cks/kexports/`，**不在本目录** |
| `MANIFEST.md` | 每份面的**来源可追溯表**（仓库/目录/build 号/符号数/是否走 `[回退]`），自动生成、勿手改；**仅在面集合或符号数变化时刷新**（"生成时间"= 内容最后变化时刻） |
| `und_unresolved_baseline.txt` | 官方模块 UND 未解析符号**基线**（自检用，见下） |

`<两位版本>` = DSM 大版本去点（7.4 → `74`），与 target 名的第二段一致（`TARGET_RE` 的捕获组）。

- 平台数：17，与 `.github/matrix/build.json` 的平台集合一一对应。当前官方内核面 47 份
  （7.0/7.1 待 ainas-dsm 补齐后重跑），官方模块面 63 份。
- 历史遗留的 `exports_vmlinux_epyc7002_74`（无 `.txt` 后缀）已由 Task#196 的自动化统一为
  `exports_epyc7002_74.txt`（内容逐字节一致）并删除，门禁的无后缀回退分支同步简化。

## 生成方式（自动化，Task#196）

由 `ainas-build/.github/workflows/kexports.yml` 触发 `ainas-modules/scripts/export_kexports.sh` 生成：

1. **官方内核**：`myp015/ainas-dsm`（旧名 arc-dsm）分支 `new` 的
   `dsm/<型号>/<DSM版本目录>/zImage` —— 版本取 7.0.1 / 7.1.1 / 7.2.2 / 7.3.2 / 7.4.1，每平台一个型号；
   目录名带 build 号（如 `7.4.1-90080-0`）故按**前缀匹配**。**尚无该版本目录时跳过、不失败**。
   **次版本内回退**：精确版本目录缺失时，回退到同一 DSM 次版本的其他 build（`7.4.*-*`，取 build 号
   最大者 —— 如 PAS7700 只有 `7.4.0-101205-0`）。同一 DSM 次版本内 kver 与内核 ABI 相同，正是门禁
   关心的粒度；回退时日志打 `[回退]` 并标注实际使用的目录名。
2. **cks 内核**：`myp015/ainas-cks` 的 `release/<平台>-<版本>-<kver>/bzImage` —— 经 `gh api`
   按需下载（不 checkout 1.4G 的 `release/`）。
3. **官方模块**：本仓库 `official/{hda1,rd}/<平台>-<版本>-<kver>/*.ko`，判据只取 `__ksymtab_<name>`
   （真 EXPORT_SYMBOL），与门禁 `nm_batch()` 完全一致。

提取链：`extract-vmlinux`（解 zImage/bzImage）→ `extract_exports_v2.py`（解析 `__ksymtab`：4.x 为 16B
绝对指针条目，5.x 为 12B prel32 条目，从 `__ksymtab_strings` 读符号名）。
⚠ `readelf` 拒绝管道（`'/dev/stdin' is not an ordinary file`）⇒ vmlinux 必须先落盘再提取。

## 自检（kexports.yml 内，Task#196）

1. **面文件健康度**（`export_kexports.sh`，硬失败）：符号数 ≥ 2000、必须含哨兵符号
   （`printk` / `kmalloc_caches` / `register_netdev` / `eth_type_trans`）、不得低于上一版同名文件的
   90%。防的是"python 提取成功但产出空/残缺面"静默通过 —— 先写临时文件、三项检查全过再 `mv`。
2. **官方模块 UND 覆盖**（`check_official_und.py`，基线比对制）：官方模块的 UND 应能在
   「内核面 ∪ 官方模块面」里解析。⚠ 官方包**自带残缺** —— `mii_*` 一族在官方 vmlinux 里根本不存在
   （实测 0 次出现），却仍打包了引用它们的 `usbnet.ko` / `8139cp.ko`；部分平台没有 `libfc.ko`
   却打包了 `tcm_fc.ko` ⇒ **不能硬门禁**，改为基线比对：只有出现**新的**未解析符号才失败
   （那才是提取口径退化或面文件损坏）。基线 `und_unresolved_baseline.txt`（当前 13 个符号）。
3. **三来源零产出**：官方内核面 / cks 面 / 官方模块面任一 `OK=0` 即硬失败
   （教训：cks 面曾因 sparse-checkout 剪掉输入而静默 0/0）。
4. **MANIFEST 幂等**：`gen_manifest()` 比对时**忽略"生成时间"行** ⇒ 面无变化就保留原文件，
   推送步骤报 `无变更`。否则每次重跑都会造一个"只改时间戳"的假 commit，
   **"无变更"这一幂等信号随之失效**（首跑实测：47+12+63 份面字节零差异，仅时间戳 1 行 diff）。

## 已知局限（门禁的兜底判据正是为它而设）

导出面按 target 的 DSM 版本选取（`gate_release.py:load_kexports(..., ver)`）；**缺该版本的面文件时
回退 `_74` 并告警**。回退期间会产生**跨 kver 假阳性**：4.4.302 backport 过 `__sk_receive_skb`，并把
`mii_ethtool_gset/sset/link_ok/nway_restart` 换成了 `mii_ethtool_get/set_link_ksettings`，4.4.180 内核
没有这些变化。

因此 `gate_release.py` 启用**兜底判据**：若某符号出现在**同一 target 的官方模块（`official/hda1`/
`official/rd`）的 UND 里**，则认为该内核提供它，从缺口里剔除。

⚠ **该判据的前提不总成立**（Task#196 实测）：官方模块的 UND 里含有**真·无人提供**的符号
（`mii_*` 与 `libfc` 两族，见上文"官方包自带残缺"）⇒ 这类剔除会**漏报真缺口**。
故剔除集现在**按面是否回退分类输出**：

- `[已知有损]` —— 面走了 `_74` 回退，剔除属跨 kver 假阳性，**补齐该版本面后应消失**；
- `[需关注]` —— 面是**精确版本**却仍剔除，只可能是官方包固有残缺、面文件不完整、或判据误判。

参见 `tasks/lessons.md` 教训 79 与 Task#196。
