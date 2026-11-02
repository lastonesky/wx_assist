# 微信助手（旧版修复）

**是什么**：2016 年豌豆荚「微信助手」1.0.6 的本地修复版，用于浏览/导出微信图片视频、清理缓存。原版已停止维护，更新服务已失效。

**时间**：原版 2016；本修复 2026-11。

**新增**：
- Android 5–10：无 root，走原版逻辑
- Android 11–17：root 桥（KernelSU/Magisk），零复制读取微信内部数据
- 一次 TREE 扫描、微信 8.0.x 新旧目录兼容
- WXGF/HEVC 图片解码、日期修复、字号调整、隐藏失效升级入口
- 外部媒体副本改为内部 symlink，启动自愈
- 清理统计适配微信 8.x；收藏夹不参与删除

**要求**：
- Android 5.0 – 17
- Android 11+ 需要 root
- Android 14+ 安装需 `--bypass-low-target-sdk-block`（示例见下）
- 微信 8.0.x（能登录服务器的版本）；7.x 不专门适配

**Android 14+ 安装示例**：

```bash
adb install -r -g --bypass-low-target-sdk-block "微信助手-改造版.apk"
```

Android 14 以下可去掉 `--bypass-low-target-sdk-block`。

**无恶意代码的最简证明**：
- 未新增任何权限、联网/上传逻辑或后台服务；新增代码只做本地文件访问和 root 文件桥
- APK 未加固/未混淆，可直接 jadx 反编译核对；完整改动见 `docs/改造与维护手册.md`
- 原始包一并保留，可对比；核心功能可断网使用

```text
微信助手.apk        sha256 44a15034ec50a9fc6131050ddb5b9038118d5911ce30615b1621765058568f74
微信助手-改造版.apk  sha256 9884b748502bfbfa2c94e771798c53c536eddc95e258dc63ae806840b399aa24
```
