title: 搭建 IPsec VPN 服务
author: Figthing
tags:
  - software
  - vpn
categories:
  - software
  - vpn
date: 2020-02-05 17:24:00
---
### 搭建 IPsec VPN 服务

#### 概要

我们在开发的时候经常遇见启动服务过多，内存消耗过大，安全问题，等，让我们非常头痛，特别是对Spring Cloud微服务的开发。在这里有同学会说把这些都部署到局域网的服务器上就行了，但如果资源有限的时候，我们对资源的利用率应该有所考虑。

将部分资源部署到云服务器，搭建一个的VPN服务，并隔离外网，这样做能让我们很好的解决上面的问题。

<!--more-->


#### 搭建&运行

**下载ipsec-vpn-server**

```shell
docker pull hwdsl2/ipsec-vpn-server
```

**配置环境变量**

这个 Docker 镜像使用以下几个变量，可以在一个 `vpn.env` 文件中定义

```
VPN_IPSEC_PSK=your_ipsec_pre_shared_key
VPN_USER=your_vpn_username
VPN_PASSWORD=your_vpn_password
```

这将创建一个用于 VPN 登录的用户账户，它可以在你的多个设备上使用[*](https://github.com/hwdsl2/docker-ipsec-vpn-server/blob/master/README-zh.md#重要提示)。 IPsec PSK (预共享密钥) 由 `VPN_IPSEC_PSK` 环境变量指定。 VPN 用户名和密码分别在 `VPN_USER` 和 `VPN_PASSWORD` 中定义。

支持创建额外的 VPN 用户，如果需要，可以像下面这样在你的 `env` 文件中定义。用户名和密码必须分别使用空格进行分隔，并且用户名不能有重复。所有的 VPN 用户将共享同一个 IPsec PSK。

```
VPN_ADDL_USERS=additional_username_1 additional_username_2
VPN_ADDL_PASSWORDS=additional_password_1 additional_password_2
```

**注：** 在你的 `env` 文件中，**不要**为变量值添加 `""` 或者 `''`，或在 `=` 两边添加空格。**不要**在值中使用这些字符： `\ " '`。一个安全的 IPsec PSK 应该至少包含 20 个随机字符。

所有这些环境变量对于本镜像都是可选的，也就是说无需定义它们就可以搭建 IPsec VPN 服务器。详情请参见以下部分。

**运行**

```
docker run \
    --name ipsec-vpn-server \
    --env-file ./vpn.env \
    --restart=always \
    -p 500:500/udp \
    -p 4500:4500/udp \
    -d --privileged \
    hwdsl2/ipsec-vpn-server
```

端口是固定的，建议不做修改，阿里云安全组中必须要增加UDP（500和4500）端口。

#### 配置 IPsec/L2TP VPN 客户端

##### Win10

1. 右键单击系统托盘中的无线/网络图标。
2. 选择 **打开网络和共享中心**。或者，如果你使用 Windows 10 版本 1709 或以上，选择 **打开"网络和 Internet"设置**，然后在打开的页面中单击 **网络和共享中心**。
3. 单击 **设置新的连接或网络**。
4. 选择 **连接到工作区**，然后单击 **下一步**。
5. 单击 **使用我的Internet连接 (VPN)**。
6. 在 **Internet地址** 字段中输入`你的 VPN 服务器 IP`。
7. 在 **目标名称** 字段中输入任意内容。单击 **创建**。
8. 返回 **网络和共享中心**。单击左侧的 **更改适配器设置**。
9. 右键单击新创建的 VPN 连接，并选择 **属性**。
10. 单击 **安全** 选项卡，从 **VPN 类型** 下拉菜单中选择 "使用 IPsec 的第 2 层隧道协议 (L2TP/IPSec)"。
11. 单击 **允许使用这些协议**。选中 "质询握手身份验证协议 (CHAP)" 和 "Microsoft CHAP 版本 2 (MS-CHAP v2)" 复选框。
12. 单击 **高级设置** 按钮。
13. 单击 **使用预共享密钥作身份验证** 并在 **密钥** 字段中输入`你的 VPN IPsec PSK`。
14. 单击 **确定** 关闭 **高级设置**。
15. 单击 **确定** 保存 VPN 连接的详细信息。

**注：** 在首次连接之前需要**修改一次注册表**。请参见下面的说明。

另外，除了按照以上步骤操作，你也可以运行下面的 Windows PowerShell 命令来创建 VPN 连接。将 `你的 VPN 服务器 IP` 和 `你的 VPN IPsec PSK` 换成你自己的值，用单引号括起来：

```
# 不保存命令行历史记录
Set-PSReadlineOption –HistorySaveStyle SaveNothing
# 创建 VPN 连接
Add-VpnConnection -Name 'My IPsec VPN' -ServerAddress '你的 VPN 服务器 IP' -L2tpPsk '你的 VPN IPsec PSK' -TunnelType L2tp -EncryptionLevel Required -AuthenticationMethod Chap,MSChapv2 -Force -RememberCredential -PassThru
# 忽略 data encryption 警告（数据在 IPsec 隧道中已被加密）
```

##### 故障排查

**Windows 错误 809**

> 错误 809：无法建立计算机与 VPN 服务器之间的网络连接，因为远程服务器未响应。这可能是因为未将计算机与远程服务器之间的某种网络设备(如防火墙、NAT、路由器等)配置为允许 VPN 连接。请与管理员或服务提供商联系以确定哪种设备可能产生此问题。

要解决此错误，在首次连接之前需要修改一次注册表，以解决 VPN 服务器 和/或 客户端与 NAT （比如家用路由器）的兼容问题。运行以下命令。**完成后必须重启计算机。**

```
REG ADD HKLM\SYSTEM\CurrentControlSet\Services\PolicyAgent /v AssumeUDPEncapsulationContextOnSendRule /t REG_DWORD /d 0x2 /f
```

另外，某些个别的 Windows 系统配置禁用了 IPsec 加密，此时也会导致连接失败。要重新启用它，可以运行以下命令并重启。

```
REG ADD HKLM\SYSTEM\CurrentControlSet\Services\RasMan\Parameters /v ProhibitIpSec /t REG_DWORD /d 0x0 /f
```

**Windows 错误 628 或 766**

> 错误 628：在连接完成前，连接被远程计算机终止。

> 错误 766：找不到证书。使用通过 IPSec 的 L2TP 协议的连接要求安装一个机器证书。它也叫做计算机证书。

要解决这些错误，请按以下步骤操作：

1. 右键单击系统托盘中的无线/网络图标。
2. 选择 **打开网络和共享中心**。或者，如果你使用 Windows 10 版本 1709 或以上，选择 **打开"网络和 Internet"设置**，然后在打开的页面中单击 **网络和共享中心**。
3. 单击左侧的 **更改适配器设置**。右键单击新的 VPN 连接，并选择 **属性**。
4. 单击 **安全** 选项卡，从 **VPN 类型** 下拉菜单中选择 "使用 IPsec 的第 2 层隧道协议 (L2TP/IPSec)"。
5. 单击 **允许使用这些协议**。选中 "质询握手身份验证协议 (CHAP)" 和 "Microsoft CHAP 版本 2 (MS-CHAP v2)" 复选框。
6. 单击 **高级设置** 按钮。
7. 单击 **使用预共享密钥作身份验证** 并在 **密钥** 字段中输入`你的 VPN IPsec PSK`。
8. 单击 **确定** 关闭 **高级设置**。
9. 单击 **确定** 保存 VPN 连接的详细信息。

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/software/vpn/image-20200205172131332.png" alt="image-20200205172131332" style="zoom: 67%;" />

参考文献：

- https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/clients-zh.md
- https://github.com/hwdsl2/setup-ipsec-vpn