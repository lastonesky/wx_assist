# 架构与知识手册 — 零复制实时桥

> 配套 `PROGRESS.md`。本文沉淀本次逆向/改造的全部技术决策与踩坑。

---

## 1. 总体架构

```
┌─ 微信助手 App进程 (uid 10282, untrusted_app_25) ────────┐
│ MmApplication.attachBaseContext                         │
│   ├─ su -c /data/local/tmp/rescue.sh   (v3纯诊断,自愈)   │
│   └─ sh files/appprobe.sh             (App身份探针)      │
│                                                         │
│ 扫描/清理/加载 改道:                                     │
│   File.listFiles/exists/isDirectory/length/lastModified │
│   File.delete / BitmapFactory.decodeFile                │
│        ↓ invoke-static                                   │
│   BridgeFS (classes2.dex, 7.7KB)                        │
│     · HashMap 缓存 {path → type,size,mtime}             │
│     · synchronized 单线程串行 (LOCK)                     │
│     · 懒加载 ensure(): Runtime.exec("su -c sh ...")      │
│     · 兜底: 桥失败/白名单外 → 普通 File API (App自有目录) │
└──────────────┬──────────────────────────────────────────┘
               │ stdin/stdout 管道 (无socket/无端口/零监听)
               │ 协议: 文本行命令 + READ二进制帧
┌──────────────▼──────────────────────────────────────────┐
│ root 子进程 (uid 0, u:r:ksu:s0)  wxbridge.sh            │
│   PING              → OK PONG                           │
│   TREE <path>       → 递归全部条目 ... END（首扫预取）    │
│   LIST <path>       → "类型\t大小\tmtime\t路径" ... END  │
│   READ <path>       → "DATA <n>" + n字节原始流           │
│   PULL <src> <dst>  → OK（root侧cp到App cache）         │
│   DELETE <path>     → OK | ERR <原因>（文件/空目录）      │
│ 安全: realpath白名单 + storage→raw归一 + 无eval          │
│ 生命周期: App死→stdin EOF→自动退出; 重启→开App自愈       │
└─────────────────────────────────────────────────────────┘
```

## 2. 协议与路径规则

- **归一**：App 侧路径 `/storage/emulated/0|/storage/self/primary|/mnt/user/0/primary/X`
  → 桥内 `normalize()` 换成 **`/data/media/0/X`（raw）** 再 realpath/白名单。
  必须 raw：App可见的 FUSE 视图会过滤指向 `app_data_file` 的 bind/symlink，root 走 raw 无此限制。
- **白名单**（`allow()`，realpath 后匹配，含目录本身）：
  - `/data/data/com.tencent.mm/MicroMsg` 及其下
  - `/data/media/0/Android/data/com.tencent.mm` 及其下
  - `/data/media/0/tencent/MicroMsg` 及其下
- **LIST 实现**：`find -L <dir> -mindepth1 -maxdepth1 -print0 | xargs -0 -r stat -L -c '%F<TAB>%s<TAB>%Y<TAB>%n'`
  （**格式串必须是真实 TAB 字节**，toybox stat 不解析 `\t` 转义；
  `-L` 必须同时给 find 和 stat，否则 symlink 会被识别成普通文件）
- **TREE 实现**：同 LIST，但 `-mindepth 0` 递归整棵子树；BridgeFS 首次访问
  `/tencent/MicroMsg` 子树时一次预取，建立 `CHILDREN`（目录→子项）缓存，
  后续 3.7 万次目录查询全部命中内存。空目录也必须建空列表。
- **不能使用** `find ... -exec stat {} +`：toybox 在大目录树会漏条目
  （image2 54257 项只返回 14744 项）。必须走 `xargs`。
- **READ 帧**：`DATA <n>\n` + 紧跟精确 n 字节；App 侧 BufferedInputStream 先按行读头、再定量读体。
- **PULL**：`PULL <src> <dst>`：`src` 走 normalize+realpath+白名单；`dst` **白名单写死**为
  `/data/data/com.wandoujia.ymir/cache/wxplay/` 下的文件（拒绝 `..`），桥端 `cp -f` + `chmod 0644`。
  这是视频点播的唯一写入点，且只能写进 App 自己的 cache（不能当任意写原语）。
  为何不用 READ：视频动辄几十 MB，走管道既占内存（MAX_READ 96MB）又在 UI 线程上慢；
  root 侧 `cp` 是内核级拷贝，换片基本瞬时。
- **视频 fd**：`setDataSource(FileDescriptor)` 的 fd 必须是**可 seek 的普通文件**；
  framework 会 dup 给 media 进程，所以调用后可立即 `close()`。
- **DELETE**：文件用 `rm -f`，空目录用 `rmdir`；非空目录由 BridgeFS.deleteTree 递归子项后再删。
  清理 Tab 的 `d.a(String,FilenameFilter)` 已改接 `BridgeFS.deleteTree`。
- **无 root 原版分支**：`BridgeFS.LOCAL_MODE = Build.VERSION.SDK_INT <= 29`；
  Android 5.0–10 直接走原版 File API（无需 root），Android 11+ 走桥。
- **symlink 模式**：外部 image2/video/voice2/favorite/avatar 可指向 `/data/data/...` 内部原件；
  桥用 `find -L` 遍历、`stat -L` 取元数据，App 仍看到 `/storage/emulated/0/...` 路径。
- **wxgf/HEVC**：图片内容以 `wxgf` + HEVC 裸流保存，BridgeFS 解析 SPS、用 MediaCodec 解码，
  再 YUV420 -> NV21 -> Bitmap。`lastModified` 把 stat 秒转换成毫秒。
- **路径含 tab 的鲁棒性**：第4段起整段为路径（split limit 语义），文件名无换行是前提（微信文件名=md5，安全）。

## 3. smali 手术清单（33处 + 视频链 3处，已固化在 `apktool_out/`）

| 类别 | 位置 | 改法 |
|---|---|---|
| 解码×3 | `android/support/v4/app/d.smali`（`d.a(String)`/`d.b(String)` 体内） | `BitmapFactory.decodeFile` → `BridgeFS.decodeFile`（描述符完全相同，纯owner替换） |
| 文件大小×1 | `manager/d.smali:222` 调用点 | `d.a(File)J` → `BridgeFS.length(File)J` |
| 危险File调用×29 | manager/{d,TrashScanner,MmConfig,ab,n,k,m,a}.smali | `invoke-virtual File->M(参)返` → `invoke-static BridgeFS->M(File,参)返`，**描述符必须把 receiver 补为首参** |
| 启动注入 | `MmApplication` | su执行块 + 探针 + RESCUE Log.i + **循环出口 `if-ge → :post_loop`**（原代码注入段是死代码的bug修复） |
| 视频点播×1 | `video/VideoPlayCtrl.smali`（`a(String)` 体内） | `setDataSource(String)` → `BridgePlay.open()` + `setDataSource(FileDescriptor)`，失败回退 |
| 播放器生命周期×2 | `video/VideoPlayView.smali` | 新增 `onDetachedFromWindow`(release) / `onWindowVisibilityChanged`(pause) |
| 排除 | manager/w,x | App自有目录（缓存mkdirs/导出），保持原生File |

- **加载链**（图片查看器）：`TouchImageView → d.a(String) → BitmapFactory.decodeFile` ← 我们改在 d 体内
- **播放链**（视频）：`VideoPlayView → VideoPlayCtrl.a(String) → setDataSource(String)` ← 已改为 `BridgePlay.open() + setDataSource(fd)`
- **列表链**：`d.a(String)根目录 → listFiles(filter n:len>20) → 递归 walk → 模型`
- **删除链**：照片/视频 `k.java → file.delete()`；清理分类无删除实现（只扫描显示）
- **`ImageUtil` 的解码点是死代码**（无外部调用者）—— 未动
- 手术脚本：`~/tools/wx_assist/workspace/bridge/patch_scan.py`（幂等，含残留/描述符校验）

## 4. FUSE / 权限能力模型（本机实测结论）

| 主体 | /sdcard共享 | Android/data | /data/data(700) | 备注 |
|---|---|---|---|---|
| App(uid10282, legacy授权) | ✅ | ❌ | ❌ | LEGACY_STORAGE+READ/WRITE appops=allow |
| adb shell / run-as | ✅ | ✅ | ❌ | **runas_app 域连 /sdcard 都拒**(dontaudit) —— 测App行为必须用App真身份探针 |
| root(ksu域) raw路径 | ✅ | ✅ | ✅ | |
| root 经 FUSE(/storage/...) | ✅ | ✅ | ❌(bind/symlink指向app_data_file被**按标签**拒) | 所以桥必须归一到raw |

- **bind mount**：目标在存储内(external) → FUSE放行 ✓；目标是 `/data/data` → 拒（`app_data_file` 标签）
- **symlink**：FUSE 层拦截/隐藏指向内部的链接 ❌（App侧不可用；**root raw 解析 symlink 没问题**）
- **结论**：App 想读内部数据只有三选一：复制(已弃) / 桥(现方案) / chcon改标签(未验证,备选)

## 5. Root 能力模型（回答过的问题沉淀）

- root 是**进程能力**不是设备开关：权限属于 `su` fork 出的子进程，不传染给 App 进程
- App 进程无法运行中提权（uid/域在 zygote 定死）→ 扫描逻辑无法"自带root"
- root 只覆盖内核层(DAC+多数SELinux)；**管不着第三方服务策略**（FUSE网关按自己规则拒）
- KernelSU 按 uid 授权；`/system/bin/su` 仅对已授权 uid 可见可执行
- Shizuku = adb/shell 权限的代理 → 实测读不了 `/data/data` → 对本场景**天花板不够**
- 通信方式选型：shell-FIFO(0体积) / Go(7-10MB,被否) / Rust(~1MB) / **stdio管道(选定: 零监听零二进制)**，
  shell 性能实测：批量 stat 管线 ≤1s ✓，逐文件 fork 415s ❌（写法决定生死）

## 6. 踩坑全记录（重跑必读）

1. **Python 在 Windows 文本模式写文件会转 CRLF**（叠出 `\r\r\n`）→ mksh 语法崩
   → 脚本一律 `open(f,'wb')` 字节写 / 或 `newline='\n'`
2. **工具链 JSON 转义吃掉 `\\t`** → python replace 变 no-op（假装成功）
   → 用 `chr(92)`/`chr(9)` 构造，杜绝转义
3. **BRIDGE 常量带 `;` 又拼 `;->`** → `BridgeFS;;->` 双分号 → apktool 组装失败
   → 常量拼接约定：`BRIDGE + "->method"`
4. **instance→static 改调用，描述符必须补 receiver 为首参**，否则运行期 VerifyError
5. **注入 smali 撞上循环出口跳转** → 注入段成死代码（表现为"完全没反应"）→ 检查所有跳转目标
6. **`/data/local/tmp` = shell_data_file**，untrusted_app 读不了 → App 执行的脚本必须放 App 私有目录或由 su(root) 读
7. **KSU 授权跟 uid 走**：卸载重装 uid 变(10146→10282) → 授权失效需重新授
8. **toybox 特性**：stat 不解析`\t`；cat 无 `-A`；无 setpriv/setfacl；cp 支持 `-n/-u` ✓；
   **`find -exec {} +` 在大目录树会漏条目 → 必须用 `find -print0 | xargs -0 -r stat`**；
   xargs `-r` 实测可用
9. **realpath 不穿越挂载点/链接链**（Android上 `/storage/emulated/0` 原样返回）→ 白名单必须自己做 normalize
10. `adb install -r` 同签名保 uid；**`--bypass-low-target-sdk-block` 绕 Android17 兼容模式**（不用改targetSdk）
11. `Runtime.exec("a b c")` 无引号语义 → `su -c sh /path` 的分词坑；全路径 `/system/bin/su` 更稳
12. uiautomator dump/截图前先 `KEYCODE_WAKEUP`（屏幕会自动锁，tap 会打空）
13. **`File`/`BitmapFactory` 能改道，`MediaPlayer.setDataSource(String)` 不能**：
    开文件的是 App 自己的 uid，桥接不进去；必须换 `setDataSource(fd)`。
14. **原版播放器没有任何暂停/释放调用点**：能播之后必须自己补 View 生命周期钩子，
    否则返回列表音频继续响（改造前因播不了而潜伏）。
15. **Windows 下 apktool 解码写 CRLF**：先批量转 LF 再 diff/patch，否则全文件 diff、exact match 失败。
16. **线上 `classes2.dex` ≠ 本地 `bridge/src/BridgeFS.java`**：改桥客户端以
    `apktool_out/smali_classes2/BridgeFS.smali` 为准；新增功能请另开类（新 dex），
    不要为了一个方法去重编既有 dex（本地旧稿已移到 `bridge/src_stale/`）。

## 7. 性能/度量数据（本机实测）

| 项 | 值 |
|---|---|
| 全树条目 / 目录 | 81831 / 约 3.7 万 |
| TREE 全量预取 | 约 1.2s |
| 旧逐目录 LIST | 数分钟仍未完成 |
| 照片 / 视频计数 | 12107 / 733 |
| image2 / video 文件数 | 30616 / 1826 |
| 外部副本策略 | 可选迁移为内部 symlink |
| 桥dex体积 | ~11KB（含预取缓存） |
| 微信内部库 | EnMicroMsg.db 等位于 `/data/data/com.tencent.mm/MicroMsg/bdcf.../` |
| 微信版本 | 8.0.79 (versionCode 3200, targetSdk 34) |
| 空间释放 | 取决于原外部副本大小 |
| PC微信4.0解密 | 29/29库, 主密钥=PBKDF2-SHA512(master,salt,256000) |

## 8. 环境与工具链（Arch Linux 原生, 在 tools/）

- `jre21-linux/` (Temurin JRE 21.0.13+9, Linux x64, 无javac→用ecj)
- `ecj.jar` (Eclipse编译器, 替代javac)
- `build-tools/android-37.0` (d8: lib/d8.jar, apksigner: lib/apksigner.jar)
- `platform35/android-35/android.jar` (编译期bootclasspath)
- `apktool.jar 3.0.3` / `uber-apk-signer 1.3.0` / `jadx 1.5.6`(只读分析)
- adb 使用系统 `android-tools` 包的原生 `/usr/bin/adb`
- 签名: `~/tools/wx_assist/workspace/rescue.keystore` (alias=rescue, storepass=android)
- 环境: `source ~/tools/wx_assist/tools/env.sh`；一键重建: `~/tools/wx_assist/tools/build_apk.sh`
- Rust: `~/.cargo/bin`（rustc 1.99.1），rustup/crates 均走 `rsproxy.cn`
- Windows 版 JRE/adb/env.bat 已删除
