# laurus 内核构建说明（SukiSU）

本仓库用于给 laurus（4.14.117 内核）构建集成 SukiSU Ultra 的内核镜像。

## 版本固定

| 项 | 值 |
|---|---|
| 内核分支 | `laurus-q-oss-ci`（4.14.117） |
| SukiSU Ultra 代码 | **`v4.1.2`**（固定 tag，不要跟 `main`） |
| 集成模式 | `builtin` |
| **管理器 APK** | **SukiSU Ultra Manager `v4.1.2`** |
| KPM | 关闭（4.14 不支持，`kpm.c` 用了 5.x 的 `access_ok`） |
| SUSFS | 启用，`SUS_PATH` / `SUS_MOUNT` / `SUS_SU` / `OPEN_REDIRECT` / `SUS_MAP` / `TRY_UMOUNT` 关闭 |

> 内核里的 KSU 版本与管理器版本必须同一条线。`v4.1.2` 内核请使用 **v4.1.2 的管理器 APK**，不要装 4.2.x 的管理器（会提示内核版本不匹配或功能异常）。

## 为什么卡在 v4.1.2

`v4.1.3` 起内核侧做了一次目录级重构，不是小版本更新：

- 结构：平铺文件 → `core/ feature/ hook/ infra/ manager/ policy/ runtime/ selinux/ sulog/ supercall/`
- hook 机制：从 kprobe 改为 **patch `sys_call_table` + patch LSM hook 槽位 + 符号解析**
- 删除：`ksu.c`、`manual_su.c`、`syscall_hook_manager.c`、`util.c`、`supercalls.h`
- 新增：`hook/syscall_hook.c`、`hook/lsm_hook.c`、`hook/arm64/patch_memory.c`、`infra/symbol_resolver.c`、`infra/event_queue.c`、`feature/selinux_hide.c`、`sulog/*`、`supercall/*`

这套新机制依赖的内核 API 全是 5.x 才有的：

| 依赖 | 内核引入版本 |
|---|---|
| `syscall_fn_t` | 4.17 |
| `security_hook_heads` 改为 hlist | 5.7 |
| `copy_from_user_nofault` / `strncpy_from_user_nofault` | 5.8 |
| fsnotify inode event（`handle_inode_event`） | 5.9 |
| `iopoll` / `remap_file_range` / `fadvise` / `mmap_supported_flags` | 4.20 ~ 5.x |
| `access_ok` 两参数（KPM） | 5.0 |

所以在 4.14 上，4.1.3 及之后的版本要大量降级（`syscall_fn_t`、LSM hook、`file_wrapper`、`seccomp_cache`、`pkg_observer` 逐项 stub），功能会残缺。**v4.1.2 是 4.14 上最后一个完整可用的版本**。

`main` 分支当前已经是 4.2.x 代码线，**不要再跟随 `main` 构建**。

## 构建方式

workflow：`build-laurus-sukisu`（文件 `.github/workflows/laurus_sukisu.yml`，位于 `laurus-q-oss-ci` 分支）

```bash
gh workflow run build-laurus-sukisu -R tanranv5/laurus --ref laurus-q-oss-ci \
  -f kernel_branch=laurus-q-oss-ci \
  -f sukisu_ref=v4.1.2 \
  -f sukisu_mode=builtin \
  -f fortify=off \
  -f build_dtbo=off \
  -f kpm=off
```

说明：

- `sukisu_ref` 默认值已固定为 `v4.1.2`
- 脚本里传给 SukiSU `setup.sh` 的参数必须是 `$ref`（tag），传 `$mode` 会导致 checkout 失败、静默回退到 `main`
- 产物为 AnyKernel3 可刷包与 `boot.img`

## 管理器

安装 **SukiSU Ultra Manager v4.1.2** 的 APK，与内核版本对齐。管理器版本高于内核时可能提示需要升级内核，版本低于内核时可能缺少对应功能入口。
