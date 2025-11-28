*作者：UpSGE*

### 写在前面

本文主要介绍以下内容：

1. ViaVersion系列（[ViaVersion](#viaversion（fabric/spigot/velocity）)、[ViaBackwards](#viabackwards（fabric/spigot/velocity）)、[ViaRewind](#viarewind（fabric/spigot/velocity）)、[ViaFabric](#viafabric（fabric）)、[ViaFabricPlus](#viafabricplus（fabric客户端）)、[ViaForge](#viaforge（forge客户端）)、[ViaAAS](#viaaas)、[ViaProxy](#viaproxy)、[ViaBungee](#viabungee（bungeecord）)、[ViaSponge](#viasponge（sponge）)、[ViaAprilFools](#viaaprilfools（fabric/spigot/velocity）)、[ViaLegacy](#vialegacy（spigot/velocity）)、[ViaRewindLegacySupport](#viarewindlegacysupport（spigot/velocity）)）；
2. 其他项目（[MultiConnect](#multiconnect（fabric）)、[ViaForgePlus](#viaforgeplus（forge）)、[ViaVanillaPlus](#viavanillaplus（fabric）)、[ViaBedrock](#viabedrock（spigot/velocity）)）；
3. Geyser系列（[Geyser](#geyser（fabric/neoforge/spigot/velocity/bungeecord/viaproxy）)、[Floodgate](#floodgate（fabric/spigot/bungeecord/velocity）)、[ThirdPartyCosmetics](#thirdpartycosmetics)、[Hurricane](#hurricane（fabric/neoforge/spigot）)、[GeyserConnect](#geyserconnect)）。

## 开始！

### [ViaVersion](https://www.mcmod.cn/class/5760.html)（Fabric/Spigot/Velocity）

- ViaVersion自身的作用是支持新版客户端连接旧版服务器（服务端和客户端都是）
- ViaVersion及其附属只翻译了网络包，有很大概率触发服务器反作弊
- ViaVersion在作为Fabric Mod时，需要[ViaFabric](#viafabric（fabric）)作为前置
- 1.20.x及以下版本支持BungeeCord和Sponge，1.21.x由[ViaBungee](#viabungee（bungeecord）)和[ViaSponge](#viasponge（sponge）)支持

### [ViaBackwards](https://www.mcmod.cn/class/5762.html)（Fabric/Spigot/Velocity）

- ViaBackwards自身的作用是支持1.9及以上的客户端连接至新版的服务器
- ViaBackwards需要[ViaVersion](#viaversion（fabric/spigot/velocity）)/[ViaFabric](#viafabric（fabric）)作为前置

### [ViaRewind](https://www.mcmod.cn/class/5761.html)（Fabric/Spigot/Velocity）

- ViaRewind自身的作用是支持1.7.X/1.8.X的客户端连接至更高版本的服务器
- ViaRewind在1.10.x及以上服务器需要[ViaBackwards](#viabackwards（fabric/spigot/velocity）)作为前置

### [ViaFabric](https://www.mcmod.cn/class/3327.html)（Fabric）

- ViaFabric在Fabric客户端及服务端实现了[ViaVersion](#viaversion（fabric/spigot/velocity）)
- ViaFabric可作为[ViaVersion](#viaversion（fabric/spigot/velocity）)的前置也可单独起作用，当[ViaFabric](#viafabric（fabric）)并未及时更新时，可以通过更新[ViaVersion](#viaversion（fabric/spigot/velocity）)来更新

### [ViaFabricPlus](https://www.mcmod.cn/class/9446.html)（Fabric客户端）

- ViaFabricPlus是面向客户端的[ViaFabric](#viafabric（fabric）)和[MultiConnect](#multiconnect（fabric）)替代品，可连接至几乎所有版本（包括正式版、Beta版、Alpha版、Classic版、战斗测试8C、部分愚人节版、最新快照版本和最新基岩版）的Minecraft服务器
- 相比[ViaVersion](#viaversion（fabric/spigot/velocity）)，ViaFabricPlus更深层地修改了客户端代码，几乎不会触发服务器反作弊
- ViaFabricPlus支持连接最新的基岩版服务器，这通过[ViaBedrock](#viabedrock（spigot/velocity）)实现，但缺少部分功能
- ViaFabricPlus支持连接Beta版、Alpha版、Classic版等，由[ViaLegacy](#vialegacy（spigot/velocity）)实现

### [ViaForge](https://www.mcmod.cn/class/5728.html)（Forge客户端）

- ViaForge是面向Forge的[ViaVersion](#viaversion（fabric/spigot/velocity）)客户端实现
- ViaForge支持连接以下版本的服务器

        正式版 1.0.0 - 1.21.8
        Beta 版 b1.0 - b1.8.1
        Alpha 版 a1.0.15 - a1.2.6
        Classic 版 c0.0.15 - c0.30

### [ViaAAS](https://github.com/ViaVersion/VIAaaS)

- ViaAAS是一个独立的[ViaVersion](#viaversion（fabric/spigot/velocity）)代理服务，能无缝地将不同版本的Minecraft客户端连接到同一台后端服务器
- ViaAAS还拥有一个便捷的网页认证系统，确保在线模式下的安全连接。

### [ViaProxy](https://github.com/ViaVersion/ViaProxy)

- ViaProxy是一个独立的[ViaVersion](#viaversion（fabric/spigot/velocity）)代理服务，允许玩家加入每个Minecraft服务器版本（由[ViaLegacy](#vialegacy（spigot/velocity）)支持）
- 允许加入联机模式服务器和Minecraft Realms，支持[Simple Voice Chat](https://www.mcmod.cn/class/3693.html)
- 有图形界面

### [ViaBungee](https://hangar.papermc.io/ViaVersion/ViaBungee)（BungeeCord）

- 1.21.x的[ViaVersion](#viaversion（fabric/spigot/velocity）)BungeeCord实现

### [ViaSponge](https://modrinth.com/plugin/viasponge)（Sponge）

- 1.21.x的[ViaVersion](#viaversion（fabric/spigot/velocity）)Sponge实现

### [ViaAprilFools](https://www.mcmod.cn/class/16366.html)（Fabric/Spigot/Velocity）

- ViaAprilFools需要[ViaBackwards](#viabackwards（fabric/spigot/velocity）)作为前置
- 支持连接到这些版本的服务端：

        3D Shareware
        20w14infinite
        战斗测试 8C

- 支持这些版本的客户端加入：

        3D Shareware
        战斗测试 8C

### [ViaLegacy](https://github.com/ViaVersion/ViaLegacy)（Spigot/Velocity）

- 在[ViaProxy](#viaproxy)和[ViaFabricPlus](#viafabricplus（fabric客户端）)中提供以下版本支持

        Classic (c0.0.15 - c0.30 including CPE)
        Alpha (a1.0.15 - a1.2.6)
        Beta (b1.0 - b1.8.1)
        Release (1.0.0 - 1.7.10)

### [ViaRewindLegacySupport](https://hangar.papermc.io/ViaVersion/ViaRewindLegacySupport)（Spigot/Velocity）

- ViaRewindLegacySupport是[ViaRewind](#viarewind（fabric/spigot/velocity）)的插件，为1.7.x-1.8.x修复了一些特性

### [MultiConnect](https://www.mcmod.cn/class/3293.html)（Fabric）

- 一个用于连接多个Minecraft服务器版本的模组，允许新版本的Minecraft上的客户端连接到较早版本（以及当前版本）上的服务器
- 已停更

### [ViaForgePlus](https://www.mcmod.cn/class/14638.html)（Forge）

- 只支持1.8.9
- ViaForgePlus还原了高版本客户端的动画和移动方式（如1.14+的1.5格潜行通过）
- 针对客户端发包进行了修复，避免了跨版本造成的反作弊误判

### [ViaVanillaPlus](https://www.mcmod.cn/class/12665.html)（Fabric）

- ViaVanillaPlus用于处理Vanilla+类型模组不同版本之间的网络协议更改，可作为[ViaFabricPlus](#viafabricplus（fabric客户端）)的附属使用
- 需要[ViaVersion](#viaversion（fabric/spigot/velocity）)/[ViaFabric](#viafabric（fabric）)作为前置
- 目前支持模组：

        Carpet
        Syncmatica（共享原理图）

### [ViaBedrock](https://github.com/RaphiMC/ViaBedrock)（Spigot/Velocity）

- ViaBedrock作为ViaVersion插件添加对MCBE服务器的支持
- ViaBedrock处于非常早期的开发阶段，尚未打算定期使用

### [Geyser](https://www.mcmod.cn/class/9757.html)（Fabric/NeoForge/Spigot/Velocity/BungeeCord/ViaProxy）

- Geyser可以让基岩版客户端加入Java版的服务器
- 当前支持基岩版1.21.70 - 1.21.101与Java版1.21.7 - 1.21.8间的连接
- 基岩版客户端可选装[GeyserOptionalPack](https://geysermc.org/download/?project=other-projects&geyseroptionalpack=expanded)资源包来修复一些特性

### [Floodgate](https://geysermc.org/download/?project=floodgate)（Fabric/Spigot/BungeeCord/Velocity）

- Floodgate让基岩版客户端能够使用基岩版帐户加入Java版服务器
- 能够在Java版上查看基岩版玩家皮肤
- 允许基岩版玩家使用全局链接或本地链接链接到Java账户

### [ThirdPartyCosmetics](https://geysermc.org/download/?project=other-projects&thirdpartycosmetics=expanded)

- Geyser的拓展，能让Java版加载第三方饰品

### [Hurricane](https://geysermc.org/download/?project=other-projects&hurricane=expanded)（Fabric/NeoForge/Spigot）

- 修复了竹子和滴水石的碰撞
- 需要[Geyser](#geyser（fabric/neoforge/spigot/velocity/bungeecord/viaproxy）)作为前置

### [GeyserConnect](https://geysermc.org/download?project=other-projects&geyserconnect=expanded)

- GeyserConnect是Geyser的一个扩展，它允许您使用单个GeyserMC代理加入多个服务器

## 推荐配置

### 跨版本

#### 单一服务端

- 根据需要安装Via系列Mod、插件

#### 群组服

- 只在代理端（推荐Velocity）安装Via系列插件

#### 客户端

- 安装[ViaFabricPlus](#viafabricplus（fabric客户端）)
- 按需安装[ViaVanillaPlus](#viavanillaplus（fabric）)

### 跨平台

- [Geyser](#geyser（fabric/neoforge/spigot/velocity/bungeecord/viaproxy）)+[GeyserOptionalPack]()+[ThirdPartyCosmetics](#thirdpartycosmetics)+[Hurricane](#hurricane（fabric/neoforge/spigot）)