# 微信助手拯救计划 — 进度与恢复手册

> 更新时间：P2/P3 完成（TREE 预取、WXGF 解码、P3 symlink），微信 8.0.79
> 交付目录：`/home/ted/Projects/wx_assist`（只保留 docs + 原始APK + 改造APK）
> 工程/工具目录：`~/tools/wx_assist/`（原 `E:/re/wx_assist`，已迁移 Arch Linux）
> 本文档是断点续传入口 —— 回家后按「恢复步骤」逐条执行即可继续推进。

---

## 0. 一句话现状

APK 已在 Android 17（澎湃OS4，KernelSU）上安装运行，**零复制实时桥方案已上线**：
App 扫描/清理/图片加载全部改道经 `BridgeFS → stdio 管道 → root 侧 wxbridge.sh`。
**当前：照片 12109 / 视频 733，清理 798.77MB，图片查看器正常，日期/字号/设置已修。**
全树 81831 条目通过一次 `TREE` 预取，实测约 1.2 秒；P3 已删除 4.8G 副本并改为 symlink。

---

## 1. 阶段总表

| 阶段 | 内容 | 状态 |
|---|---|---|
| A | APK 修复：删32位.so、8个native方法Java桩、debuggable、`--bypass-low-target-sdk-block` 安装、smali死代码bug(`:post_loop`) | ✅ 完成 |
| B | root 通道：uid 10282 授 KSU；App 启动 `su -c rescue.sh`；PC侧 `adb shell run-as + su -c` | ✅ 完成 |
| C-旧 | 数据桥 v1（复制方案）：4.8G 内部数据复制到外部 + bind mount | ✅ 已被桥方案替代；P3 已删除副本 |
| C-新 | **零复制实时桥**（当前方案） | ✅ P2/P3 完成 |
| D | 端到端验证 | 🔄 扫描/清理/图片/日期/HEVC 已通过；删除链待安全测试 |

### C-新 细分
| 步骤 | 状态 | 说明 |
|---|---|---|
| P0 本地构建 | ✅ | bridge 脚本+BridgeFS+d8+smali手术33处+apktool构建+multidex注入+签名，全部本地验证 |
| P1 上机 | ✅ | 推脚本、rescue改纯诊断、测试文件、装APK、桥进程确认拉起、零崩溃 |
| P2 UI验证 | ✅ | 照片 12109 / 视频 733，清理 798.77MB，HEVC/日期/字号/设置已修 |
| P3 删副本 | ✅ | 4.8G 副本已删除；image2/video/voice2/favorite/avatar 已改为内部 symlink |
| P4 收尾 | ⏳ | 手机端临时文件去留、文档定稿 |

---

## 2. 设备当前状态（手机侧已存在的东西）

```
uid=10282 (u0_a282)  com.wandoujia.ymir  已授予 KernelSU root（uid 重装会变！变后需重新授权）
安装: ~/tools/wx_assist/workspace/bridge/signed/bridge-p0-inj-aligned-signed.apk (覆盖安装, rescue.keystore 签名, minSdk21, multidex)
/data/local/tmp/wxbridge.sh          ← root桥服务 (stdio协议, 含 TREE, find -L, stat -L)
/data/local/tmp/rescue.sh            ← v3纯诊断 (已停止sync!)
/data/local/tmp/sync_all.sh          ← 旧同步脚本(已不被调用, P4可删)
/data/local/tmp/appprobe.sh 等       ← 旧调试脚本(P4可删)
/data/data/com.wandoujia.ymir/files/{appprobe.sh,rescue.sh,app_probe.log}  ← App私有探针
/sdcard/tencent/MicroMsg/            ← 合成树:
    ├── Cache/ xlog/ bdcf.../{image,music,sns}   ← bind mount (指向外部微信数据)
    ├── bdcf.../{image2,video,voice2,favorite,avatar} ← ★symlink -> /data/data/... 内部原件
    ├── CheckResUpdate / WebviewCache            ← symlink -> 内部缓存 (微信8.x)
    ├── Cache/{android_data,magicPath,mediaOpt,magicSclCover} ← symlink -> 新版缓存
    ├── browser/ weixin/ (微信旧残留)
    └── BRIDGE_DELETE_TEST.txt (删除链测试用)
/sdcard/rescue.log                   ← 启动诊断日志
微信内部目录权限: 已恢复 700/600 (隐私OK, 桥以root直读不需要chmod)
```

**P3 已完成**：image2 2.2G + video 2.3G + voice2 269M + favorite + avatar ≈ 4.8G
外部副本已删除，改为指向 `/data/data/com.tencent.mm/MicroMsg/bdcf.../` 的 symlink。
桥端 `find -L` + `stat -L` 跟随 symlink，App 无感知；释放约 4.8G。
微信 8.0.79 新增缓存通过 `~/tools/wx_assist/workspace/bridge/wechat8_cache_links.sh` 映射进旧清理分类，不复制数据。

---

## 3. ✅ P2 修复记录（计数归零 + 扫描性能）

### 3.1 计数归零：listFiles 多跳一个 TAB

`BridgeFS.listFiles()` 解析桥返回行时，原代码从第 1 个 TAB 开始又跳了 **3 次**，
落到不存在的第 4 个 TAB，导致每个条目被丢弃 → 扫描结果恒为空。

修复：只再跳 **2 次**，正确落在第 3 个 TAB 后的 path 上。

### 3.2 扫描慢：逐目录 LIST 往返

微信目录树实测 **81831 条目 / 约 3.7 万目录**。旧协议一次只列一层，
App 递归扫描会发 3.7 万次 `LIST`，每次都启动 shell/find/stat → 数分钟。

修复：
- `wxbridge.sh` 新增 `TREE <dir>`：一次递归输出整棵子树
- `BridgeFS.prefetchTree()`：首次访问 MicroMsg 子树时一次拉全量，
  同时建立 `CHILDREN`（目录 → 直接子项）缓存
- 空目录也必须预建空列表，否则空目录仍会 cache-miss 走 LIST
- 后续所有 `listFiles()` 直接命中内存，零桥往返

实测：`TREE` 81831 行 / 36848 个目录，约 **1.2 秒**；之后桥 I/O 基本归零。

### 3.3 toybox 隐藏坑：find -exec {} + 漏条目

Android toybox 的 `find ... -exec stat {} +` 在大目录树上会漏结果：
image2 实际 54257 项，只返回 14744 项。

修复：桥端统一改为 `find -print0 | xargs -0 -r stat`，完整无漏。

### 3.4 验证结果（Android 17 实机）

| 项 | 结果 |
|---|---|
| 照片 Tab | 12107 张 |
| 视频 Tab | 733 个 |
| 清理 Tab | 微信垃圾 45.38MB / 朋友圈缓存 2.10KB / 收藏夹 2.92MB / 头像 15.65MB / 朋友圈图片视频 26.81MB |
| 图片查看器 | 正常打开并显示 |
| logcat | 无 VerifyError / FATAL / BridgeFS cache-miss |

数量差异说明：App 只统计带扩展名的媒体文件，image2 中大量 `th_*` 无扩展名缩略图不计入。

### 3.5 P3 + UI/兼容性修复（已完成）

- **P3**：删除 4.8G 外部副本，image2/video/voice2/favorite/avatar 改为 symlink；
  桥端 `find -L` + `stat -L` 已修复 symlink 被识别成普通文件的问题。
- **日期 1970**：`BridgeFS.lastModified()` 原先直接返回 `stat %Y` 秒，App 按毫秒解析；
  已改为 `* 1000`。照片按内部真实 mtime 分组，降序排列。
- **灰色照片**：约 80% 图片是微信 `wxgf`（HEVC 裸流伪装 .jpg）。
  已实现 SPS 解析 + MediaCodec HEVC 解码 + YUV->Bitmap，缩略图正常显示。
- **设置**：`检查升级` 容器设为 `gone`，保留 view 避免 NPE。
- **字号**：`text_size_normal 13->15`、small `12->14`、mini `11->13`、大标题/卡片同步放大。
- **清理适配（微信 8.0.79）**：`wechat8_cache_links.sh` 把
  `Android/data/com.tencent.mm/cache`、`CheckResUpdate`、`webview_tmpl`、
  `magicPath`、`mediaOpt`、`magicSclCover` 映射进旧分类；
  清理总量从 45.38MB 提升到 **798.77MB**（微信垃圾 756.31MB）。
- **收藏夹已排除**：`favorite` 是用户数据不是缓存，已隐藏并从清理总量/删除中排除。
- **无 root 分支**：`BridgeFS.LOCAL_MODE = SDK<=29`，Android 5.0–10 直接走原版 File API，不启动 su；Android 11+ 走桥。
- **清理删除链已接桥**：`BridgeFS.deleteTree()` + smali 补丁
  `android/support/v4/app/d.a(String,FilenameFilter)`；桥 DELETE 支持空目录 rmdir，
  非空目录由 App 递归子项后删除。**尚未对真实缓存执行清理按钮**。

**修完后的重建管线（Arch Linux 原生，照抄即可）**：
```bash
cd ~/tools/wx_assist
source ~/tools/wx_assist/tools/env.sh

# 1)-4) 编译 BridgeFS -> d8 -> apktool 打包 -> 注入 classes2.dex -> 签名
~/tools/wx_assist/tools/build_apk.sh --clean

# 5) 推送桥脚本 + 安装（当前阶段已获准直接安装）
adb push ~/tools/wx_assist/workspace/bridge/wxbridge.sh /data/local/tmp/wxbridge.sh
adb shell chmod 666 /data/local/tmp/wxbridge.sh
adb install -r -g --bypass-low-target-sdk-block \
    "$HOME/tools/wx_assist/workspace/bridge/signed/bridge-p0-inj-aligned-signed.apk"
```

> `~/tools/wx_assist/tools/build_apk.sh` 已封装 5 步管线；smali 手术仍固化在 `~/tools/wx_assist/workspace/apktool_out/`，
> 不需要每次重跑 `patch_scan.py`。

**smali 手术**（33处）已固化在两处，**不要重复执行**（脚本幂等但无必要）：
- 改动本体：`~/tools/wx_assist/workspace/apktool_out/smali/`
- 可重跑脚本（含校验）：`python ~/tools/wx_assist/workspace/bridge/patch_scan.py`（若 apktool_out 被还原才需要）

**P2 验证清单**：
1. ✅ 照片/视频计数非0（12109 / 733）
2. ✅ 清理 Tab 出现大小（798.77MB；微信垃圾 756.31MB；收藏夹已排除）
3. ✅ 点开照片 → 查看器显示（含 wxgf/HEVC）
4. ⏳ 删除测试：需在不破坏用户媒体的前提下验证 `BridgeFS.delete`
5. ✅ logcat 无 VerifyError/崩溃/cache-miss/JNI abort

---

## 4. 其余待办（P2 尾部 / P3 / P4）

- [ ] **视频播放**：`VideoPlayCtrl:533 MediaPlayer.setDataSource(String)` 单点未接桥。
      方案候选（需用户拍板）：a) fd流式（零写入, 实现最难）b) 按次临时文件（用户点播才写, 量级=单个视频）
      c) 保留 video 副本（现已是 symlink，回退需重新复制）
- [ ] **语音播放**：同视频，未勘察（可能也是 setDataSource）
- [x] **P3 删副本**：已完成（2026-11-02）；备份/校验见 `~/tools/wx_assist/workspace/bridge/p3_symlink.sh`
- [ ] **删除链测试**：`BRIDGE_DELETE_TEST.txt` 或安全测试素材
- [x] **清理删除实现**：已接 BridgeFS.deleteTree；需用户确认后再实际点“清理”
- [ ] **清理按钮实跑验证**：会删除约 800MB 缓存，执行前确认
- [ ] **P4**：删手机端旧调试脚本（/data/local/tmp/*.sh 除 wxbridge.sh；App私有目录探针去留）；
      本地目录再清理一轮；文档定稿；rescue 自愈重建 symlink

---

## 5. 恢复步骤（回家后）

```bash
# 0) 工具环境（Bash 原生，不再需要 Git Bash/Wine）
cd ~/tools/wx_assist
source ~/tools/wx_assist/tools/env.sh

# 1) 手机连接 + 状态检查（系统原生 adb，android-tools 包）
adb devices
adb shell "pidof com.wandoujia.ymir; ps -A | grep wxbridge"

# 2) root 通道（PC侧万能钥匙, 已验证）
adb shell "run-as com.wandoujia.ymir sh -c 'su -c id'"
# 若 uid 变了(重装导致) → 让用户在 KernelSU 里删旧条目重新授权

# 3) 桥协议独立测试(不经App)
printf 'PING\nTREE /storage/emulated/0/tencent/MicroMsg\n' > /tmp/t.txt
adb push /tmp/t.txt /data/local/tmp/t.txt
# ... 见 workspace/bridge/bridge_cmds.txt + run_bridge_test.sh (模式照抄)

# 4) 重建 → 推桥脚本 → 装机 → 验证清单
#    详见 §3.4 的重建管线
```

**注意**：屏幕会自动锁 → 每轮 UI 测试前 `adb shell input keyevent KEYCODE_WAKEUP`；
uiautomator dump 可用性不稳，必要时截图目测。

### Linux 构建环境（当前机器）

已切换到 Arch Linux 原生工具链，**不再使用 Wine / Windows JRE / adb.exe**：

```bash
cd ~/tools/wx_assist
source ~/tools/wx_assist/tools/env.sh

# JAVA_HOME -> tools/jre21-linux (Temurin JRE 21.0.13+9, Linux x64)
# PATH       -> jre21-linux/bin + tools/jadx/bin；adb 用 /usr/bin/adb

# 一键重建（ecj -> d8 -> apktool -> 注入 -> uber-apk-signer）
~/tools/wx_assist/tools/build_apk.sh --clean

# Rust（可选，当前仅安装备用）
rustc --version   # 1.99.1；rustup/crates 走 rsproxy.cn
```

原生工具均已在 Arch 上实跑验证：`apktool 3.0.3`、`jadx 1.5.6`、`d8 9.2.4`、
`uber-apk-signer 1.3.0`、`apksigner 0.9`、`ecj 3.39.0`、`adb 37.0.0`。
Windows 版 JRE/adb 已删除，不再保留。
若 adb 无权限识别设备，安装 `android-udev` 并重新插拔/重启 adb server。

---

## 6. 关键文件索引

| 路径 | 内容 |
|---|---|
| `docs/ARCHITECTURE.md` | 桥协议、smali手术清单、FUSE/root 能力模型、全部踩坑记录 |
| `~/tools/wx_assist/workspace/bridge/wxbridge.sh` | root 桥服务（唯一真源, 含 TREE/find -L/stat -L） |
| `~/tools/wx_assist/workspace/bridge/rescue_v4.sh` | 启动自愈：诊断 + P3 symlink 检查 + 微信8缓存链接 |
| `~/tools/wx_assist/workspace/bridge/p3_symlink.sh` | P3 删除副本/建 symlink 的一次性脚本（含校验） |
| `~/tools/wx_assist/workspace/bridge/wechat8_cache_links.sh` | 微信 8.x 缓存映射（CheckResUpdate/WebviewCache/Android cache） |
| `~/tools/wx_assist/workspace/bridge/src/.../BridgeFS.java` | App 桥客户端（含 TREE 预取 + 目录缓存） |
| `~/tools/wx_assist/workspace/bridge/patch_scan.py` | smali 批量手术脚本（幂等+校验） |
| `~/tools/wx_assist/workspace/bridge/bridge_cmds.txt` + `run_bridge_test.sh` | 协议独立测试套件 |
| `~/tools/wx_assist/workspace/apktool_out/` | 修改后 APK 工程（smali 已含全部手术 + RESCUE 日志 + :post_loop修复） |
| `~/tools/wx_assist/workspace/bridge/signed/*.apk` | 最终可安装产物 |
| `~/tools/wx_assist/workspace/bridge/wxbridge.phone-before-tree.sh` | P2 前手机端旧桥脚本备份 |
| `~/tools/wx_assist/workspace/rescue.keystore` | 签名密钥（重打包必须, alias=rescue, pass=android） |
| `~/tools/wx_assist/tools/` | jre21-linux, jadx, apktool, ecj, d8(build-tools37), android.jar(platform35) |
| `~/tools/wx_assist/tools/env.sh` | Linux 原生工具链环境变量（JAVA_HOME/ANDROID_JAR/D8_JAR/...） |
| `~/tools/wx_assist/tools/build_apk.sh` | 一键重建 APK（编译→d8→apktool→注入→签名，不自动安装） |
| `~/tools/wx_assist/archive/signed9/` | 旧"副本方案"APK（回滚备份） |
| `~/tools/wx_assist/wx-decrypt/` | PC微信4.0解密工具与产物（独立成果） |

---

## 7. 流程红线（用户明确要求，永远有效）

1. **对手机的一切写入/修改/删除，先列清单说明影响，拿到明确"可以"再动手**
2. PC 侧新工具链下载同样先报备（本次已获批：build-tools + platform35 + ecj）
3. 微信数据默认只读；删除只发生在用户于 App 内明确操作 或 已批准的 P3/P4 清理
4. 每步保留回滚路径（`~/tools/wx_assist/archive/signed9` = 副本方案；`~/tools/wx_assist/workspace/apktool_out` = 任意版本可重建）
