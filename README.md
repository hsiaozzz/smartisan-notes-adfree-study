# 锤子便签 4.3.0 去广告改造研究

基于锤子便签 4.3.0（versionCode 43000）APK 的修改与实机验证记录。本次发布的是可与原版并存的独立研究测试包，包含广告入口禁用、安装身份与数据入口隔离，以及主页书本按钮的跳转修复。

**仅供个人学习研究，请勿用于商业用途。** 本项目与原应用开发者无关联；原应用、商标与第三方组件的权利属于各自权利人。本仓库不对原应用或第三方代码授予开源许可。

## 下载与安装

前往 [Releases](https://github.com/hsiaozzz/smartisan-notes-adfree-study/releases) 下载 `SmartisanNotes-adfree-test-clone.apk`。APK 放在 Release 附件中，仓库保存改造说明和验证资料。

| 项目 | 本次发布 |
| --- | --- |
| 桌面名称 | Smartisan Notes Ad-free Test |
| 安装包名 | `com.smartisan.notes.adfree.test` |
| 应用版本 | 4.3.0 / 43000 |
| 最低系统 | Android 5.0 / API 21（兼容性未逐版本验证） |
| 目标系统版本 | API 30 |
| 架构 | arm64-v8a、armeabi-v7a |
| 签名 | Android 调试证书，v1 / v2 / v3 |
| 已测试设备 | Huawei OXF-AN10，Android 12 / API 31 |

该包使用独立的应用身份和数据空间，可与原版 `com.smartisan.notes` 并存。本次测试没有卸载或清空原版，也没有迁移原版便签。测试包不会自动拥有原版账号、便签或登录状态。

```bash
adb install -r SmartisanNotes-adfree-test-clone.apk
adb shell am start -W -n com.smartisan.notes.adfree.test/com.smartisan.notes.Splash
```

改造后无法使用原应用的签名。即使保持原包名，也不能直接覆盖官方签名版本，因此这里发布已通过实机测试的独立包。请先备份重要数据，不要为安装本包而直接卸载原版。

## 改造过程

### 1. 分析 APK 与广告调用链

使用 Apktool 解码 Manifest、资源和 8 个 DEX 的 Smali，结合调用关系查找广告初始化、开屏、插屏、激励视频和推荐卡片入口。分析确认 Meishu 广告 SDK 与聚合的 Pangle、GDT 相关组件存在。

修改方式是在应用侧关键方法入口提前返回，保留 SDK 类、资源和 Manifest 组件，避免直接删除依赖导致缺类或资源错误。下表中的名称来自 APK 反编译结果，部分已经混淆。

| 文件 / 类 | 修改的方法 | 行为 |
| --- | --- | --- |
| `smali_classes2/nf1.smali` | `OooO00o(Landroid/content/Context;)V` | 提前返回，禁用此处 Meishu 初始化 |
| `smali_classes6/com/smartisan/notes/Splash.smali` | `OoooOoO(Lgn;Z)Z` | 返回 `false`，阻止开屏广告分支 |
| `smali_classes6/vv3.smali` | `OooOOO(Landroid/app/Activity;Lgn;Z)V` | 提前返回，禁用 InsertSplashController 入口 |
| `smali_classes5/com/smartisanos/notes/ad/OooO00o.smali` | `OooO0oO()V` | 提前返回，阻止推荐广告卡片请求 |
| `smali_classes4/com/smartisan/notes/sync/MoreActivity.smali` | `o0000O0()V` | 提前返回，保持激励广告设置行隐藏 |
| 同上 | `o00O0O()Lgt9;` | 直接返回 `gt9.OooO00o`，结束相关回调 |
| `smali_classes4/com/smartisan/notes/sync/OooOOOO.smali` | `OooOOOO(Lut2;)V` | 提前返回，禁用激励视频点击入口 |

`@id/ad_switch` 的布局默认是 `gone`，本次改造阻止代码重新显示该设置行。修改后检查正常导航和回调，不让界面停留在等待广告结果的状态。

这些改动禁用已定位的广告入口，**不代表 APK 中全部第三方组件、遥测或网络请求都被删除**，也没有绕过登录、会员或支付检查。

### 2. 构建原包名版本并校验

先完成广告入口补丁，使用 Apktool 重建，再执行 zipalign 和 apksigner。检查签名、ZIP 完整性，并再次反编译核对补丁。

由于手机上已经安装官方签名版本，修改包不能直接覆盖它。为保留原版数据，后续实机测试改用独立安装包名；原包名构建产物不是本次 Release 的附件。

### 3. 适配与原版并存

将安装包名改为 `com.smartisan.notes.adfree.test`，桌面名称改为 `Smartisan Notes Ad-free Test`。Java / Smali 类名保留原来的命名空间，避免全局替换破坏类引用。

- 调整 Manifest 中的 Provider authorities、taskAffinity 和自定义权限名称。
- 将特殊的 `com.smartisan.knbsnote` authority 改为 `com.smartisan.notes.adfree.test.knbsnote`。
- 同步调整代码、资源 XML 中写死的 Provider 地址和读写权限，保证便签访问自身数据入口。
- 调整 `syncadapter_notes.xml` 的便签 contentAuthority。
- 为设置按钮的 `smartisanos.intent.action.more_view` Intent 指定测试包，解决并存时系统“没有应用可执行此操作”的问题。

初次并存测试曾出现 Provider authority 冲突，随后又发现写死的原版数据地址引发权限错误；上述引用修复后，本地便签功能通过。该适配主要验证本地功能，账号与云同步仍有限制。

### 4. 修复主页书本按钮

主页书本按钮原来发送未指定组件的 Intent：

```text
Action: android.intent.action.VIEW
Category: android.intent.category.DEFAULT
URI: snssdk1883://com.bd.notes/rag
```

实机点击时，系统将其交给“打开方式”选择器，显示“没有应用可执行此操作”。在 `smali_classes5/com/smartisanos/notes/NotesActivity.smali` 的 `o0000Oo0()V` 中，保留原 URI 和参数，在 `startActivity` 前明确指定应用内的知识库页面：

```smali
const-class v1, Lcom/bd/librag/RAGMainActivity;
invoke-virtual {v0, p0, v1}, Landroid/content/Intent;->setClass(Landroid/content/Context;Ljava/lang/Class;)Landroid/content/Intent;
```

更新后，书本按钮可进入“知识库广场”并加载列表，返回主页后再次进入也通过。

### 5. 重建、对齐与签名

使用的工具：Apktool 3.0.3、Android SDK Build Tools 36.0.0、Android Studio 自带 JBR、ADB。以下为示意命令，路径和签名信息需按自己的环境填写：

```bash
java -jar apktool_3.0.3.jar d input.apk -o decoded
# 按上文修改 decoded 中的 Smali、Manifest 与相关资源引用
java -jar apktool_3.0.3.jar b decoded -o unsigned.apk
zipalign -f -p 4 unsigned.apk aligned.apk
apksigner sign --ks YOUR_KEYSTORE --ks-key-alias YOUR_ALIAS \
  --v4-signing-enabled false \
  --out SmartisanNotes-adfree-test-clone.apk aligned.apk
apksigner verify --verbose --print-certs SmartisanNotes-adfree-test-clone.apk
zipalign -c -p 4 SmartisanNotes-adfree-test-clone.apk
sha256sum SmartisanNotes-adfree-test-clone.apk
```

本仓库是过程记录，不包含原应用源码、原 APK、签名私钥或自动一键修改脚本；以上命令需在完成手动补丁后使用。

## 实机验证与已知限制

2026-10-09 在 Huawei OXF-AN10 / Android 12 上完成以下检查：

- 安装与更新成功，原版仍保留。
- 新建、输入、保存、重新打开便签通过；更新和重启后数据保留。
- 冷启动和后台恢复通过，测试时未观察到开屏广告；两次冷启动约 762 ms、725 ms。
- 设置页上下部分可访问，未显示激励广告入口；便签列表未观察到推荐广告卡片。
- 修复后的书本按钮两次进入知识库广场通过，列表加载成功。
- 最后一轮本地功能测试的 crash buffer 为空，书本按钮复测日志未发现崩溃。

未登录账号，未验证云同步、会员、AI 对话、知识库写入、分享、推送或支付。日志中存在测试包账号类型注册的 `SecurityException` 和后台 `UrlUtils` 类初始化告警，未导致已测本地功能崩溃。知识库广场能够打开不代表账号与云端功能全部正常。

目前只验证上述设备、场景与入口，其他系统和使用路径可能仍有问题。详细结果见 [实机测试记录](docs/TESTING.md)。

![书本按钮修复后打开知识库广场](assets/knowledge-plaza.png)

## 校验

发布 APK 的 SHA256：

```text
308d1afcb74e7d50b0a2458d9c42fcf5efa730e2e55131f7ec9b9cd7605d32b7
```

同时提供 `SHA256SUMS` 附件，便于核对下载文件：

```bash
sha256sum -c SHA256SUMS
```
