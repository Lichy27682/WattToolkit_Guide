# Windows 系统使用 Watt Toolkit 时如何配置 Git

## 1. 背景介绍

Git 工具本身并不需要加速器或代理，但当我们完成本地操作后需要与远程服务器交互时，就会需要一些魔法来稳定网站的访问。作为轻量且免费的软件，[Watt Toolkit](https://steampp.net/) (原名 Steam++ )当仁不让成为加速 GitHub 的首选。但上手之后我们会发现诸多问题并不是仅点击一下“一键加速”就能解决的。本文档作为学习笔记将记录汇总本人遇到的各种问题，并给出问题原理剖析与解决方案。

## 2. 概述

在GitHub仓库主页可以看到 `<> Code` 按钮下提供了3种 Clone 的途径，也就是3种本地仓库访问 GitHub 的途径：HTTPS、SSH 和 GitHub CLI。最后一种并不常见，因此不做过多研究。本文主要还是围绕 HTTPS 与 SSH 两种访问方式介绍，二者各有优劣，选取哪种可以阅读下文后自行判断。

使用 HTTPS 连接，即 使用你的 GitHub 账号名称与密码、以 GitHub 的网址作为目标去连接服务器，相比于 SSH 而言看似门槛更低，实则出现的问题更多样、更繁琐、更复杂，连接更不稳定，更不安全。其胜在与 VS Code、GitHub Desktop 的图形界面契合度更高，操作更简便，适合个人工作场景使用。

使用 SSH 连接，即 使用已经配置好的的公钥与私钥、以 GitHub 的 ip 地址为目标去连接服务器。配置有一点门槛，与图形界面的结合也不尽人意，需要手敲的命令行较多；但配置流程十分规范化，可能出现的问题非常有限，因此整体流程下来相比 HTTPS 反而更容易一些。不仅如此，使用密钥对连接相比 SSL 协议认证更安全可靠，连接更稳定，往往被公司企业等有特殊需求的多人合作工作场景所采用。

## 3. 原理与流程

---------

### HTTPS

HTTP 主要由两部分组成：HTTP + SSL/TLS，也就是在 HTTP 上又加了一层处理加密信息的模块。服务端和客户端的信息传输都会通过 TLS 进行加密，所以传输的数据都是加密后的数据。可以看出相比于 HTTP，HTTPS 的安全基础在于 SSL 协议，因此在连接过程中若遇到 SSL 证书问题<mark>**并不推荐使用关闭 SSL 认证的方法掩耳盗铃**</mark>，否则会存在中间人攻击的风险。

<a id="watt-cer"></a>

顺便一提，Watt Toolkit 的证书可以通过在其软件界面内打开 `网络加速 -> 加速设置 -> 打开证书文件夹 -> SteamTools.Certificate.cer` 找到。为方便表述，**我们不妨将其文件地址记为 `path\of\watt\certificate`**。

Git 的后端 SSL 认证有2种，分别是使用 **OpenSSL** 与使用 **schannel**：
- OpenSSL 是一个开放源代码的跨平台软件库包，能同时在 Linux、MacOS、Windows 多平台上运行。应用程序可以使用这个包来进行安全通信，避免窃听，同时确认另一端连接者的身份。
- schannel 是安全支持提供程序 (SSP)、可实现安全套接字层 (SSL) 和传输层安全 (TLS) Internet 标准身份验证协议。为 Windows 专属，仅能在 Windows 上运行。

而两者进行 SSL 认证的方式也会不同。**schannel 会读取 Windows 系统的根证书库认证**；而 **OpenSSL 仅会读取其自身的独立证书文件，不会读取系统证书库。**

由于Git最初是为了Linux系统而设计的，因此为了跨平台考虑，Git for Windows 也总是默认使用 OpenSSL 作为 SSL 后端，这就会导致仅同步到系统证书库的新证书并不会被OpenSSL读取到，从而造成SSL认证缺失。

|组件|证书读取方式|是否读取 Windows 证书库|
|:---:|:---:|:---:|
|Chrome/Edge|Windows 证书库|✅ 是|
|Git (OpenSSL)|独立证书文件|❌ 否|
|Git (schannel)|Windows 证书库|✅ 是|

鉴于 Watt Toolkit 提供的证书并不在 OpenSSL 独立证书文件中，因此直接加速并尝试 git 指令大概率会遭遇 SSL 认证缺失报错。我们可以采用以下几种手段解决：

#### 方案A：显式指定证书路径

可以通过 `git config` 指令强制指定 git 的证书读取路径。复制前面获取的 [watt 证书路径](#watt-cer) 并使用命令行：
```
git config --global http.sslCAInfo "path\of\watt\certificate"
git config --global http.sslVerify true
```
即可设置路径。

<mark>**评价**</mark>：该方法无需更改证书文件，安全性好。但其存在一个严重的问题是，一旦设置成功则 git 只会使用该路径读取证书，原本 OpenSSL 证书文件和系统根证书库的文件都将无法被读取。这意味着此后使用 git 能且仅能使用 Watt Toolkit 加速器连接 GitHub，更换甚至不开启加速器都会报错 SSL 认证缺失。因此仅推荐每次使用 git 连接远程仓库时必定开启 Watt Toolkit 的人使用该方案。

#### 方案B：临时配置环境变量

环境变量是另一种传递配置的方式，优先级高于 `git config`。当 git 启动时会读取该环境变量指向的证书文件，直接覆盖默认证书路径，效果等同于 `sslCAInfo`。这个方法适合临时测试或不想修改全局 git 配置的场景。

可以在命令行输入：
```
export GIT_SSL_CAINFO="path\of\watt\certificate"
```
或者直接自己在 `高级系统设置 -> 环境变量 -> 系统变量(或用户变量也可)` 中新建变量：
```
变量名：GIT_SSL_CAINFO
变量值：path\of\watt\certificate
```

<mark>**评价**</mark>：此方法不涉及 `.gitconfig` 全局配置，但新建变量有可能会被系统清理，仅适合临时测试使用的情况。

#### 方案C：将证书合并到 Git 的默认证书文件

既然 OpenSSL 仅会读取其自身携带的证书文件，那么我们仅需将 Watt Toolkit 的证书添加进其证书文件即可在完全不变换后端、不破坏 git 全局配置的情况下一劳永逸解决问题，是一种非常优雅的解决方案[^c]。

首选我们通过命令行获取 git 使用的证书文件的路径：

```
# 1. 确认 Git 实际使用的证书文件路径与 SSL 后端
git config --system --list | findstr ssl
git config --global --list | findstr ssl
```
确保输出结果中包含 `http.sslbackend=openssl` 与 `http.sslverify=true`，否则使用命令行：
```
# 如果出现问题，恢复配置
git config --global --unset http.sslbackend
git config --global http.sslVerify true
```
**记所获证书路径 `http.sslcainfo` 为 `path\of\git\certificate`**。为防止证书损坏便于回档，我们先备份证书：
```
# 2. 备份原文件
Copy-Item "path\of\openssl\certificate" "path\of\openssl\certificate.backup"
```
完成后不要急着合并证书！需要先确认 Watt 的证书是否是文本格式(PEM)的，否则若是二进制格式(DER)直接追加合并会损坏 OpenSSL 的证书文件！可以通过下面方法检查：
```
# 3. 检查证书格式

# 方法1：用记事本打开，看是否显示为文本
notepad "path\of\watt\certificate"

# 方法2：使用 PowerShell 读取文件头
Get-Content "path\of\watt\certificate" -TotalCount 2

# 如果输出是乱码，说明是 DER 格式，需要转换为 PEM
# 使用 openssl 转换（Git 自带 openssl）
"Path-of-Git\Git\mingw64\bin\openssl.exe" x509 -inform DER -in "path\of\watt\certificate.cer" -out "path\of\watt\certificate.pem"
# 此时重新记转换格式后的文件地址 path\of\watt\certificate.pem 为 path\of\watt\certificate

# 如果输出已经是文本（PEM 格式），直接使用
```
确保 Watt 证书格式已经为文本格式时就可以使用命令行合并证书了：
```
# 4. 合并 Watt 证书到 OpenSSL 证书文件
Get-Content "path\of\watt\certificate" | Add-Content "path\of\openssl\certificate"

# 或者直接使用记事本打开2张证书，复制粘贴即可
notepad "path\of\watt\certificate"
notepad "path\of\openssl\certificate"
```
该指令会将 Watt 证书内容复制添加到 OpenSSL 文件的末尾。仅需输入下面的命令行比对输出结果即可：
```
# 5. 验证证书
Get-Content "path\of\openssl\certificate" -Tail 20
Get-Content "path\of\watt\certificate"
```

若两个命令行输出结果相同，则说明添加成功。但 `Add-Content` 指令仅会机械地复制粘贴证书文本，并不会根据证书信息在证书开头添加注释。因此为了方便后续查验，建议手动添加注释，方法是使用记事本打开证书文件：
```
notepad "path\of\openssl\certificate"
```
随后找到最后一段被 `-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----` 分隔的证书，不出所料其与前面一段证书之间应当没有空行与注释。我们只需将注释 `# SteamTools Certificate` 插入到最后的证书前面即可。保存后关闭即可，可通过下面命令行验证注释是否插入成功：
```
# 6. 验证注释
Select-String -Path "path\of\openssl\certificate" -Pattern "SteamTools Certificate"
```
若有返回值则说明插入成功。

<mark>**评价**</mark>：该方案兼顾灵活性、稳定性与兼容性，并且让 Git 保持兼容性更高的 OpenSSL 后端的同时，在更换或关闭加速器之后依然不影响正常的 SSL 认证，十分优雅省心。唯一的问题是该方法存在有效持续时间，到 Watt 证书有效期到期更新后就会失效，需要注意及时更新，届时直接使用记事本打开 Git 的证书文件复制粘贴手动更新即可。
```
notepad "path\of\watt\certificate"
notepad "path\of\openssl\certificate"
```

#### 方案D：使用系统代理

这个方法的核心思想是让 Git 直接走系统代理，绕过 Watt Toolkit 的 HTTPS 证书拦截。打开 Watt Toolkit 软件界面，进入 `网络加速 -> 加速设置`，在加速模式中选择 "System" 即可。

<mark>**评价**</mark>：该方案操作简单，可逆性高，但问题在于不稳定，有时可能会出现网速低甚至断连的情况，可以临时使用。

#### 方案E：切换 SSL 后端为schannel

由前面的介绍我们知道切换后端可以使 Git for Windows 读取系统根证书库，从而解决 SSL 认证的问题，但其也有着如下诸多问题：

- 已有的 OpenSSL 配置可能被静默忽略，导致连接行为改变。
- 如果企业证书已安装到系统库，schannel 正常工作；但如果没有，可能需要额外配置。
- schannel 对 TLS 版本和加密套件的支持与 OpenSSL 不同，可能在使用 Clash、v2rayN 等代理软件时出现握手失败。

```
# 切换到 Windows 证书存储后端
git config --global http.sslbackend schannel

# 确保验证开启
git config --global http.sslVerify true

# 验证配置
git config --global --list | findstr ssl
```
|特性|OpenSSL（默认）|schannel|
|:---:|:---|:---|
|证书来源|独立文件路径|Windows 系统证书库|
|跨平台|✅ 完全一致|❌ 仅限 Windows|
|自定义 CA|✅ 灵活|⚠️ 依赖系统|
|企业代理|✅ 支持|⚠️ 可能冲突|
|调试信息|✅ 详细|❌ 有限|
|性能|✅ 较快|⚠️ 稍慢|
|系统集成|❌ 不读取|✅ 自动读取|
|维护成本|需手动管理|系统自动管理|

<mark>**评价**</mark>：此方案固然能完美、便捷且持久地解决证书读取的问题，但问题却出在 schannel 后端本身上。作为 Windows 系统的专属，其性能、平台兼容性等方面均逊色于 OpenSSL，与未来为了兼容其他服务器所额外花的精力相比完全得不偿失。

#### 总结

|方案|复杂度|风险|可逆性|持久性|优点|缺点|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|方案 A（sslCAInfo）|低|低|高|持久|精确控制，最安全|灵活性差|
|方案 B（环境变量）|低|低|高|临时|快速，不改全局配置|环境变量可能被系统清理|
|方案 C（合并证书）|中|中|中|持久|一劳永逸|修改系统文件有风险|
|方案 D（系统代理）|低|中|高|持久|绕过 MITM|可能降低速度或失效|
|方案 E（schannel）|低|中|高|持久|与系统集成，最省心|某些公司 VPN 可能不兼容|

--------

### SSH

相比于 HTTPS 认证，使用 SSH 更需要一点学习门槛（虽然前面的操作也不简单就是了）。但相应地，使用 SSH 密钥连接更安全、更稳定。尤其是在多人合作办公的场景中，连接除 GitHub 外的远程服务器更是必定采用 SSH 认证，因此推荐读者学习相关技能。

#### Step 1. 创建 .ssh 文件夹

考虑到创建非默认密钥名并不会自动创建 .ssh 文件夹，因此个人推荐提前手动创建。虽然 .ssh 文件夹理论上可以随意创建到电脑的任意地方，但还是推荐读者创建到系统默认位置，这样系统能够自动读取识别。
```
# 1. 先创建 .ssh 目录
mkdir "C:\Users\你的用户名\.ssh"
```

#### Step 2. 生成密钥对

创建完成后即可使用 `ssh-keygen` 命令生成密钥对。生成密钥可以选择生成所使用的算法，如 RSA、ed25519 等。个人更推荐使用 ed25519，更现代化。

|特性	|ed25519	|RSA|
|:---:|:---:|:---:|
|**算法类型**	|椭圆曲线密码学（EdDSA）	|非对称加密（RSA算法）|
|**密钥长度**	|固定256位	|可配置（4096位）|
|**安全性**	|更高（现代密码学）	|较高（成熟但需更长密钥）|
|**生成速度**	|极快	|较慢|
|**签名速度**	|更快	|较慢|
|**文件大小**	|更小（私钥约90字节）	|较大（4096位私钥约3KB）|
|**兼容性**	|较新，但主流平台已支持	|最广泛，几乎全兼容|
|**推荐程度**	|**GitHub官方推荐**	|传统选择|

生成密钥时可以考虑自定义密钥文件名，这在管理多对密钥时至关重要。
```
# 2. 生成密钥
ssh-keygen -t ed25519 -C "your_gitconfig_email@example.com"

# 当提示输入保存路径时，若需要自定义文件名，输入：
C:\Users\你的用户名\.ssh\自定义密钥文件名
```
生成命令完成后 .ssh 文件夹中会多出两个文件，这就是密钥对。`.pub` 后缀文件是公钥，需要将其部署到服务器中；无后缀的文件是私钥，需要留在本地慎重保管，**一定一定不要将其传给其他人或上传到网络上！**

#### Step 3. （可选）创建与编写 config 文件

虽然对于仅需一对密钥使用的低需求用户而言并不需要 config 文件，但出于鲁棒性的考虑仍然建议所有使用者创建 config 文件管理密钥，以便于未来需要多对密钥时直接扩充。
```
# 3. 创建 config 文件
type nul > %USERPROFILE%\.ssh\config
# 或者在 .ssh 文件夹中直接右键创建文本文档后删除后缀名
```
创建后可以直接使用记事本打开，也可以使用其他代码编辑器打开（使用 VS Code 可以考虑安装 `Remote - SSH` 插件，能帮助管理远程服务器，以及填充 config 文件的语法）。

Config 文件的书写遵循以下格式：
- `Host` 是必需的，用于区分不同的配置块
- 参数使用缩进（空格或 Tab）表示属于该 Host
- 参数名大小写敏感，但通常使用首字母大写
- 注释以 `#` 开头
- 支持通配符 `*` 和 `?`

```
# 示例：
Host 别名
    参数1 值1
    参数2 值2
    ...
```
参数可见下面表格：

|参数	|说明	|示例|
|:---:|:---:|:---:|
|Host	|别名（必需）	|`Host github.com`|
|HostName	|真实主机名/IP	|`HostName 192.168.1.100`|
|User	|登录用户名	|`User git`|
|Port	|SSH 端口（默认 22）	|`Port 2222`|
|IdentityFile	|私钥文件路径	|`IdentityFile ~/.ssh/id_ed25519`|
|IdentitiesOnly	|只使用指定密钥	|`IdentitiesOnly yes`|
|ForwardAgent	|代理转发	|`ForwardAgent yes`|
|ServerAliveInterval	|心跳检测间隔	|`ServerAliveInterval 60`|
|StrictHostKeyChecking	|主机密钥检查	|`StrictHostKeyChecking no`|
|Compression	|压缩传输	|`Compression yes`|
|Timeout	|连接超时	|`Timeout 10`|

<mark>**注意！！！** </mark><a id="config-remark"></a>

- 当你以及配置了 config 文件并连接服务器时，系统仅会检索 config 文件中设置的别名 `Host`，而不会检索实际的主机名 `Hostname`！如二者皆需使用，请将另起一个 Host。
- 若不设置 `Port` 端口值，则其默认使用22端口。但 Watt Toolkit 会劫持域名 `github.com` 并将其解析为本地回环地址 `127.0.0.1`。但 Watt Toolkit 默认只监听443端口，因此需要指定 `Port` 的值为443。为了连接效率更高，推荐将域名 `Hostname` 也一并改成 `ssh.github.com`（SSH over 443专用服务器）。

下面是笔者个人的 config 文件配置，仅供参考：
```
# Global Default Config
Host *
    IdentitiesOnly yes


# Github account: personal
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519_github_personal

# Github account: work
Host github-work
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519_github_work
```

#### Step 4. 添加公钥到 GitHub

完成配置即可将公钥部署到 GitHub 的账号上。进入并登陆 [GitHub](https://github.com/) 之后点击 `头像 -> Settings -> SSH and GPG keys -> New SSH key`，将公钥的内容复制到 Key 这一栏，再在 title 一栏中给该密钥起名后点击 `Add SSH key` 保存即可。

#### Step 5. 连接测试

使用命令行进行测试：
```
ssh -T <User值>@<Host值>
```
例如在我上面的 config 文件的配置情况下，若要连接工作所用账号则执行：
```
ssh -T git@github-work
```
即可。若成功，则返回结果：
```
Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```

#### 补充：添加远程仓库

值得注意的是，若使用 SSH 连接 GitHub，那么仓库对象就不再是 `https://github.com/...` 网址了，否则执行远程仓库相关指令时系统仍会通过链接为网址从而判断出使用 HTTPS 进行连接。正确方式应当是在仓库主页的 `<> Code` 按钮下选择 SSH 并复制链接，随后在添加仓库时将链接中的 `github.com` 更改为你自己 config 文件中的 Host 别名即可。

--------

## 4. 常见报错及其原因

1. `ssh: connect to host github.com port 22: Connection refused`
   这表明此时电脑无法连接到 GitHub 的 22 端口。如何你使用的是 SSH 认证，那么可能你没有[使用 config 文件通过配置强行将访问端口修改为443](#config-remark)。除此之外，GitHub 的 22 端口在某些网络环境下（如公司网络、校园网、某些地区）会被防火墙或 ISP 封锁，导致连接被拒绝。若不是 config 文件的问题则可以尝试更换无线网。

2. `Git: fatal: unable to access 'https://github.com/...':SSL certificate OpenSSL verify result: unable to get local issuer certificate (20)`
   这表明系统正在使用 HTTPS 协议认证并且无法访问到正确的 SSL 证书。
   - 如果你本意就是使用 HTTPS 认证，那么说明 Watt Toolkit 的证书不在检索范围内。可能是你忘记指定证书路径，或者你指定的证书并非 Watt 真正的证书；也可能是你找错了 OpenSSL 真正的证书文件，或者证书拷贝出现了问题（如没有转换证书格式、拷贝缺失等）；若使用 schannel 后端也可能是 watt 证书没有成功加入系统证书库中。快捷键 `Win + R` 打开 `certmgr.msc` 后打开 `受信任的根证书颁发机构 -> 证书`，检查是否存在证书 `SteamTools Certificate`。如若没有，打开 Watt Toolkit 软件界面，进入 `网络加速 -> 加速设置`，点击“安装证书”尝试重新安装证书，或者重新安装软件。
   - 如果你本意是使用 SSH 认证，说明你的远程仓库链接错误使用了网址，请[更改链接为 SSH 的链接](#补充添加远程仓库)。

3. `Git: fatal: unable to access 'https://github.com/...':Recv failure: Connection was reset`
   说明你的 Watt Toolkit 未能正常工作。这常在使用[方案D](#方案d使用系统代理)中出现，也是不推荐使用该方案的原因。更换其他方案即可。

[^c]: 诚实的 DeepSeek 如此评价我独具慧眼的选择。