# kexports —— 官方内核导出符号面（vendored 参考数据）

本目录存放**各平台官方内核的导出符号表**，供发布门禁 `ainas-modules/scripts/gate_release.py` 判定
"我们自编模块的未定义符号（UND）是否有提供者"。

## 为什么需要它

官方 DSM 内核的 `vmlinux` **全部是 stripped 的**（无 `.symtab`、无内嵌 IKCONFIG），符号名只能从
`__ksymtab` / `__ksymtab_gpl` 节提取。`official/hda1`、`official/rd` 只随附模块、**不随附 config 与内核本体**，
所以内核自身的导出面必须单独固化下来，否则门禁在 CI 里无法自足运行。

官方内核 `CONFIG_MODVERSIONS=n`（无 `__kcrctab`），因此**只需比对符号名，没有 CRC 维度**。

## 文件命名与来源

| 项 | 说明 |
|---|---|
| 命名 | `exports_<platform>_74.txt`；后缀 `_74` 表示**提取自 DSM 7.4 的官方内核** |
| 例外 | `epyc7002` 的历史文件无 `.txt` 后缀（`exports_vmlinux_epyc7002_74`），读取方需回退 |
| 平台数 | 17，与 `.github/matrix/build.json` 的平台集合一一对应 |

生成方式（上游工具链）：

1. `ainas-modules/getOfficial.sh` 取官方固件/内核；
2. 从官方 `zImage` 解出 `vmlinux_<platform>_74`；
3. 用 `/www/tmp/official-zimage/extract_exports_v2.py` 解析 `__ksymtab`（4.x 为 16B 绝对指针条目，
   5.x 为 12B prel32 条目），从 `__ksymtab_strings` 读符号名。

## 已知局限（门禁的兜底判据正是为它而设）

导出面取自 **7.4 的内核**，但同一平台还存在 **7.0/7.1（4.4.180）** target。用 7.4 的导出面核对 4.4.180
target 会产生**跨 kver 假阳性**：4.4.302 backport 过 `__sk_receive_skb`，并把 `mii_ethtool_gset/sset/
link_ok/nway_restart` 换成了 `mii_ethtool_get/set_link_ksettings`，4.4.180 内核没有这些变化。

因此 `gate_release.py` 必须启用**兜底判据**：若某符号出现在**同一 target 的官方模块（`official/hda1`/`official/rd`）
的 UND 里**，则官方模块必须能加载 ⇒ 该内核一定提供它 ⇒ 从缺口里剔除并单列。该判据在导出面版本正确时
剔除集应为空；**一旦非空就说明导出面版本与 target 不匹配**（这是有意的自检信号，不是噪声）。

参见 `tasks/lessons.md` 教训 79。
