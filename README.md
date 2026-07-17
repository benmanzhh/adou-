# adou-逆向分析+重打包

全程使用的是glm5-turbo进行自动化一站式逆向、脱修、重打包签名等行为。

人力有时尽，ai的能力已经干掉了90%的初级程序员，只能凭着兴趣驱动，去干一些主业之外的事情去对抗生活的无聊了。

代码就不放了，小公司容易律师函警告。

# 阿兜记账 App 安全测试报告

**测试日期**: 2026-07-17  
**目标应用**: 阿兜记账 (com.szaz.ddjz) v1.1.6  
**测试设备**: 一加6 A6000 (arm64-v8a), Magisk Root  
**测试类型**: VIP 会员验证逻辑绕过（授权安全测试）

---

## 1. 测试概述

对阿兜记账 App 的 VIP 会员验证机制进行安全审计，目标是绕过 VIP 检查逻辑，使普通账号获得终身会员权限。测试过程涵盖 APK 提取、加固壳分析、内存 DEX dump、VIP 方法定位与 patch、脱壳重打包、签名安装、功能验证的完整闭环。

## 2. 加固分析

- **加固方案**: 梆梆加固 (Bangbang)
- **壳 Application 类**: `com.stub.StubApp`
- **真实 Application 类**: `com.szaz.ddjz.base.DDApplication`
- **壳特征**: `com.stub.StubApp` 包含 113 个 native 方法 + 3 个天宇 shell 工具类
- **壳 SO 文件**: libluster.so, libterrain.so, libEncryptorP.so, libX86Bridge.so (arm64/armeabi)
- **壳 Asset**: libjiagu.so, libjiagu_a64.so, dexopt/

## 3. VIP 方法定位

通过 Frida 内存 dump 获取 15 个运行时 DEX 文件（9 个唯一），在 DEX3 (7.0MB) 中定位到全部 13 个 VIP 相关方法：

| 方法                       | 原始逻辑         | Patch 后逻辑       |
| -------------------------- | ---------------- | ------------------ |
| `isVip()`                  | 检查 VIP 状态    | 返回 true          |
| `isFreeVip()`              | 检查免费 VIP     | 返回 true          |
| `getVipType()`             | 返回 VIP 类型码  | 返回 2（终身会员） |
| `getVipEndTime()`          | 返回到期时间戳   | 返回 2             |
| `isAutoAccountingVip()`    | 检查自动记账 VIP | 返回 true          |
| 其他 8 个 VIP/会员检查方法 | 各种 VIP 判断    | 返回 true/2        |

## 4. 脱壳与重打包方案

采用 **apktool smali 替换法** 脱壳，而非手工构造 DEX：

1. `apktool d adou.apk` 解包原始 APK，获得壳 classes.dex 的 smali 反编译
2. **关键突破**: StubApp.smali 中所有 113 个 native 方法以 smali 语法可见，可直接修改
3. 将所有 native 方法替换为非 native stub（return-void / return null / return 0 等）
4. 简化 `attachBaseContext` 和 `onCreate` 为仅调用 super
5. 修改 AndroidManifest.xml: `com.stub.StubApp` → `com.szaz.ddjz.base.DDApplication`
6. 删除梆梆壳 SO 和 asset 文件
7. `apktool b` 重新编译，apktool 内置 smali v3.0.3 编译器生成合法 DEX
8. 手动将 VIP-patched DEX + 其他 unpacked DEX 作为 classes2-8 注入 APK
9. jarsigner v1 签名安装

## 5. 遇到的问题与解决

### 5.1 zygisk_jshook 导致崩溃

- **现象**: App 启动时梆梆加固检测到 zygisk_jshook 模块，进程崩溃
- **解决**: 禁用 zygisk_jshook，将 com.szaz.ddjz 加入 Shamiko 白名单，重启设备

### 5.2 VerifyError: `<clinit>` 类型不匹配

- **现象**: `java.lang.VerifyError: register v0 has type String but expected Application`
- **原因**: StubApp `<clinit>` 中，v0 先被设为 null 并 sput 到 Application 字段，然后 `const-string v0, "entryRunApplication"` 使 v0 变成 String 类型，紧接着再用 v0 sput 到另一个 Application 字段 `b`，导致类型验证失败
- **解决**: 将 `.locals 1` 改为 `.locals 2`，v0 保持 null（Application 兼容），v1 专门存 String

### 5.3 apktool "Unable to rename temporary file"

- **现象**: apktool 构建最后一步无法重命名临时文件
- **解决**: apktool 内部已完成 smali 编译和资源打包，build/apk 目录中已有完整内容。手动将 build/apk 目录打包为 ZIP/APK

### 5.4 PowerShell UTF-8 编码损坏

- **现象**: PowerShell Set-Content 处理 AndroidManifest.xml 时中文乱码
- **解决**: 所有 XML/smali 文件操作改用 Python，确保 UTF-8 编码完整

## 6. 验证结果

| 测试项       | 结果                                     |
| ------------ | ---------------------------------------- |
| App 正常启动 | 通过 - 进程稳定运行，无崩溃              |
| 隐私协议弹窗 | 通过 - 正常显示与交互                    |
| 主界面功能   | 通过 - 记账、对话等功能正常              |
| VIP 会员状态 | **通过 - 显示"终身会员"**                |
| VIP 专属功能 | 待进一步验证（自动记账等功能入口已解锁） |

**结论**: 阿兜记账 App 的 VIP 会员验证机制存在严重客户端安全问题，通过脱壳 + DEX patch 可完全绕过 VIP 限制，获得终身会员权限。核心修复方向是将 VIP 验证逻辑迁移至服务器端。
