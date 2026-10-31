# 淡漠临时 root · 5.15.178 提权研究

> 目标设备：**iQOO Z9（vivo V2361A / PD2361 / 高通 SM7550）**
> 目标内核：`5.15.178-g74d0068f20c8-dirty`（Android 16 / OriginOS 6）
> 状态：**已打通**（临时 root，重启失效）；成功率实测约 80%（v8.9 血统）

这个仓库只放**能在 178 内核上拿到临时 root 的那条链路**的研究资料与装配脚本，
不包含 App 本体，也不包含任何第三方二进制。

---

## 1. 它是什么

一条 **page-cache 写（DirtyFrag / CVE-2026-43284）→ call_usermodehelper → root 脚本 → KernelSU LKM** 的临时提权链路。

```
① 页缓存写原语
   DirtyFrag：把只读页（apex 里的 crash_dump64 / vendor 下的 .so）在页缓存里改掉
        │
② 触发
   让系统自己去加载被改过的页（`libc++: mutex acquired, loading custom module`）
        │
③ 内核侧回调
   被改的驱动在合适时机调用 call_usermodehelper，以 **root** 拉起
   /data/user/0/<pkg>/dsh （= 本仓库 scripts/rootstage.sh）
        │
④ 用户态装配
   rootstage.sh：setenforce 0 → 找 ksud → `ksud late-load --kmi android13-5.15`
   → KSU 内核模块（LKM）装载 → 装配 /data/adb/ksu/bin → 产出可用的 su
        │
⑤ 结果
   uid=0、SELinux 域切到 su（或 permissive）、CapEff 满
   **重启即失效**（模块是 LKM，不落盘、不改 boot）
```

### 关键事实

| 项 | 值 |
|---|---|
| 内核 | `5.15.178-g74d0068f20c8-dirty`（从 197 降级固件得到） |
| BL | 锁（`verifiedbootstate=green`），不能刷第三方 boot |
| 驱动的 vermagic | 必须与 `uname -r` **逐字相同** |
| KMI | `android13-5.15` / `android14-5.15` |
| 内核配置 | `MODULE_SIG_FORCE` 未开、`MODVERSIONS=y`、`CFI_CLANG=y`、`modules_disabled=0` |
| 持久性 | 无。重启回到官方驱动状态 |

---

## 2. 仓库结构

```
docs/
  01-原理与链路.md          链路每一环的机制与失败点
  02-复现步骤.md            从零到拿到 root 的完整步骤
  03-失败模式与判据.md      12 条判据 + 常见失败对照表
  04-内核偏移与结构.md      5.15.178 的偏移表（含"错值 vs 正确值"对照）
  research/
    S18-5.15.178-提权研究归档.md     同 SoC 的独立研究（SLUB/pid_max/偏移纠错）
    Z9-5.15.178-拿root-交接文档.md   实机结论与踩坑记录
scripts/
  rootstage.sh       以 root 身份执行的装配脚本（本链路的核心胶水）
  ko_load.sh         内核模块安全加载器：vermagic 闸门 + 失败回滚
  install.sh         装机（df.r / com.danmo.df2 两种包名）
  make-symlinks.sh   包名切换时的符号链接处理
  sign-apk.sh        zipalign + apksigner + 验签一条龙
tools/
  set_vermagic.py    把 .ko 的 vermagic 改写成目标内核串（等长约束，安全）
logs/
  样例-链路日志.md    脱敏后的成功/失败日志片段（用来对照现象）
```

---

## 3. 快速开始（研究用途）

> 完整的坑位、判据和恢复步骤见 `docs/02-复现步骤.md`，这里只给最短路径。

```bash
# 0) 确认内核：不是这个串，后面全部作废
uname -r        # 5.15.178-g74d0068f20c8-dirty

# 1) 准备上游组件（本仓库不打包，见 THIRD_PARTY.md）
#    - DirtyFrag / DFRoot 的注入器与 LKM 源码
#    - KernelSU（GPL-3.0）的 ksud 与 LKM

# 2) 编/改内核模块，让 vermagic 与本机一致
python3 tools/set_vermagic.py dirtyfrag.ko "5.15.178-g74d0068f20c8-dirty SMP preempt mod_unload modversions vivo aarch64"

# 3) 跑链路（一次开机只跑一次）
#    见 docs/02-复现步骤.md

# 4) 三重验证，缺一即"假 root"
id                                              # uid=0
cat /proc/self/attr/current                     # u:r:su:s0
grep ^CapEff /proc/self/status                  # ffffffffffffffff
ls /data/adb/ksu/bin/su                         # 用户态 su 已装配
```

---

## 4. 两条硬规矩

1. **一次开机只跑一次**。失败就重启，不要连点：页缓存状态会越打越脏，第二次成功率更低。
2. **拿到 root 后不要再跑链路**。重跑会打乱已装载的驱动，root 会掉（"自毁循环"）。

---

## 5. 免责与法律

- 本项目**仅用于安全研究与自有设备取证**。在非自有设备上使用可能违反当地法律。
- 链路依赖第三方组件（DirtyFrag / DFRoot / KernelSU），本仓库**不包含**它们的源码或二进制，
  许可证与合规由使用者自行向上游确认（见 `THIRD_PARTY.md`）。
- 提权会修改内核内存，**存在 panic 与 `/data` 元数据损坏的风险**（作者本人遇到过桌面/设置重置、
  应用丢失）。跑之前请备份照片与文档。
- 本仓库不提供任何"一键工具"，也不对使用后果负责。

---

## 6. 许可证

- `scripts/`、`tools/`：**MIT**（见 `LICENSE`）
- `docs/`、`logs/`：**CC BY 4.0**
- 引用的第三方项目：各自许可证，见 `THIRD_PARTY.md`

---

## 7. 致谢

- **DONGDERS** — DirtyFrag（page-cache 写原语）与相关 LKM 工作
- **DFRoot** — UMH 驱动的 root 装配思路
- **KernelSU**（tiann 与社区）— `ksud` 与 LKM 装载链（GPL-3.0）
- 酷安社区关于 vivo 反 root 与"一发制"的经验