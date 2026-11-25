# 明日方舟 iOS 官服与 B 服双登录

<!-- Created by Xuyiming -->

同一个 iOS 客户端提供两个登录入口。玩家选择的渠道决定使用哪组配置，以及由游戏原有登录逻辑还是哔哩哔哩游戏 SDK 处理本次登录。电脑上的 Python 工具负责把桥接库 `ABLoginBridge.dylib` 和配套资源打进 IPA，手机上的桥接库负责处理渠道选择与 SDK 回调。

## 从选择渠道到登出

1. **选择渠道。** 登录页首次默认 B 服，之后记住玩家的选择。桥接库中的 `ABBootstrap` 提供 `channelMenu` 和 `selectChannel:` 方法。`ABChannelProfiles.plist` 分别保存 `official` 和 `bilibili` 两组配置，各有自己的 `AppId`、`AppKey`、`Channel` 和 `World`。
2. **官服登录。** 玩家选择官服时，桥接层沿用游戏原有的登录方法，由原客户端完成账号登录。
3. **B 服登录。** 玩家选择 B 服时，`ABLoginCoordinator` 将登录交给哔哩哔哩游戏 SDK。SDK 展示登录界面，并通过回调将结果交给协调器。库中可见登录成功、登录失败和登出三个独立入口，成功回调 `didLoginSuccessWithUid:accessKey:` 接收用户标识和 SDK 返回的凭据。
4. **重试与切换。** 按发布说明，进入游戏后渠道按钮隐藏，登出返回登录页后再次显示。玩家可以在下一次登录时改选渠道，也可以在取消或失败后重新发起登录。[上游发布说明](https://github.com/lazhire/Arknights-Bilibili-for-IOS/releases/tag/bat)

以上职责依据 2.7.91 / build 76 发布包中的渠道配置、`ABBuildInfo.json`、预编译桥接库的方法符号和作者说明整理。成功回调的存在能确认客户端接收 SDK 结果的入口。**服务端是否接受本次登录，仍需在真机上用 B 服账号验证。**

## 后续版本如何维护

发布工具按 Bundle ID、版本号和 build 号精确选择适配配置。现有配置分别支持 2.7.81 / build 71 和 2.7.91 / build 76。遇到未知 build 时，工具会停止并生成 `compatibility-*.json`。

| 游戏更新情况 | 维护工作 |
| --- | --- |
| 仅更新游戏内资源，客户端 build 和登录流程均未变 | 通常可以继续使用已安装的修补包。 |
| 发布新 build，登录接口与现有桥接库兼容 | 可以沿用双渠道登录逻辑。为新 build 单独核对宿主二进制、补丁位置和配套资源，加入匹配的版本配置，再做封装校验。随后在真机上分别验证官服登录、B 服登录、失败重试和登出切换。 |
| 游戏或 B 站 SDK 改变登录接口、回调或鉴权流程 | 需要重新确认双方的接口约定，可能要修改并重新编译桥接库。要在 fork 中独立完成这类修改，需要取得桥接库源码及对应的构建说明。目前核对到的分支和发布附件提供的是预编译库。 |

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

发布包自带的 `test_dispatch` 和 `test_profile_291` 共 15 项离线测试，覆盖版本分发、异常输入、宿主 UUID 和补丁原始字节校验。
