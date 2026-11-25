# 明日方舟 iOS 渠道修补工具的实现

<!-- Created by Xuyiming -->

这个工具把一份已解密、版本匹配的国服 IPA 打包成支持官服与 B 服登录切换的未签名 IPA。Python 脚本在电脑上选择适配版本、校验并写入组件。手机上的渠道选择和登录回调由随包分发的原生桥接库处理。

## 去哪里看实现

| 核对位置 | 实际内容 |
| --- | --- |
| [fork 的 main 源码快照](https://github.com/MIKU-MIKU-39/Arknights-Bilibili-for-IOS/tree/dbc585ead1c8dd7f8ca1abb7ffa4cb09ec5c5817) | 此快照只有项目说明。 |
| [v2.7.81 分支快照](https://github.com/MIKU-MIKU-39/Arknights-Bilibili-for-IOS/tree/c89f482fdf2a289bb5de23466f37bd04ad8cd955) 和 [v2.7.91 分支快照](https://github.com/MIKU-MIKU-39/Arknights-Bilibili-for-IOS/tree/c4e9b7e3e1a9ca56d3fbe2d1e2367d2aea89671e) | 两者的 `arkpatch_decrypted.py`、`kit/manifest.json` 和 `kit/payload.zip` 内容相同，都对应 **2.7.81 / build 71**。前一个分支将工具放在子目录，后一个放在根目录。 |
| [上游发布附件 v7.5.1](https://github.com/lazhire/Arknights-Bilibili-for-IOS/releases/tag/bat) | 附件增加了 `ipa_tool.py` 和版本索引。**2.7.81 / build 71** 使用 `engines/v75.py`，**2.7.91 / build 76** 使用 `engines/v75_291.py`，各有独立的适配清单与组件。 |

## 一份 IPA 如何生成

1. **选择版本。** 发布包的 `start.py` 要求工具根目录中恰好有一份 IPA。命令行入口 `ipa_tool.py` 则接受指定路径。`run()` 读取主应用的 `Info.plist`，扫描可识别的 Mach-O 文件，并按 Bundle ID、版本号和构建号匹配 `profiles/index.json`。`diagnose` 在找到配置后返回，`patch` 才调用对应引擎。
2. **核对输入。** 引擎的 `validate_input()` 检查应用目录、加密标记，以及清单中每个宿主二进制的架构和构建 UUID。清单要求这些文件各自只有一个 arm64 切片，因此同一 build 的通用框架也可能因额外带有 armv7 切片而被拒绝。引擎还会校验清单和组件包的 SHA-256，并核对每个待替换位置的原始字节。
3. **写入组件。** 以 build 76 为例，引擎按清单修改 `UnityFramework` 的指定区段，将 `ABLoginBridge.dylib`、渠道配置、SDK 资源、图标及配套的 `Info.plist` 写入新包。其余游戏资源从输入 IPA 复制。打包时过滤旧签名、描述文件和商店签名信息，输入文件始终以只读方式打开。
4. **校验并交付。** 引擎先写入 `.partial` 文件，重新打开后检查 ZIP CRC、文件清单、写入组件的 SHA-256，以及保留资源的大小与 CRC。通过后才将临时文件改名为 `*-unsigned.ipa`。`*.report.json` 记录文件变更和成品哈希，`compatibility-*.json` 记录版本选择与执行状态。安装前仍需另行签名。

## 手机上如何切换渠道

`ABChannelProfiles.plist` 分别保存 `official` 和 `bilibili` 两组渠道配置。根据发布包内的 `ABBuildInfo.json` 和发布说明，登录页默认选择 B 服，并记住之后的选择。选择官服会沿用游戏原有的登录逻辑，选择 B 服则通过哔哩哔哩游戏 SDK 显示登录界面并接收回调。进入游戏后隐藏渠道按钮，返回登录页后再次显示。支付入口会提示“商品下架”。[上游发布说明](https://github.com/lazhire/Arknights-Bilibili-for-IOS/releases/tag/bat)

发布附件提供的是编译好的 `ABLoginBridge.dylib`。上述交互来自构建元数据及作者说明。此次核对在 Python 3.12 下运行了附件的 `test_dispatch` 和 `test_profile_291`，15 项离线测试全部通过，覆盖版本分发、异常输入、宿主 UUID 和补丁原始字节校验。**2.7.91 的真机登录与服务端鉴权仍待验证。**
