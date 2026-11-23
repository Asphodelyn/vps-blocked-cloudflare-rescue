# VPS 的 IP 被墙之后：从买域名到接上 Cloudflare 小黄云的全流程

适合：海外 VPS 的 IP 被墙了，上面跑着节点、或者自己的网站和服务，又不能重装的人。

要求：数据库里的数据不能丢，也不重装服务器、不换机器。

标了"（未实测）"的地方，是照 Cloudflare、Caddy、3x-ui 的官方说明写的，我们自己的机器上没跑过。

---

## 先看你是哪种情况

| 你的服务器上有什么 | 按这个顺序看 |
|---|---|
| 只有节点（3x-ui），没有网站 | 第 0～4 步 → 第 7 步的 7.2 → 第 8、9 步。第 5、6 步跳过 |
| 只有网站或服务，没有节点 | 第 0～6 步 → 第 9 步。第 7、8 步跳过 |
| 节点和网站都有（我们的情况） | 从头到尾都要看，节点按第 7 步的 7.1 来 |
| 已经接好了，只是在报错 | 直接看第 6 步，或者文末的"52x 报错速查" |

---

## 第 0 步：确认是不是 IP 被墙

- 国内所有东西同时连不上，从海外（或者用 itdog 这类网站的海外节点）测试却一切正常 → 基本就是 IP 被墙。
- 别急着重装或换机器，数据都还在，只是国内访问不到了。接下来的思路就是：让国内的流量先到 Cloudflare，再由 Cloudflare 转到你的服务器。Cloudflare 的 IP 没被墙，就能绕过去。

---

## 第 1 步：买域名
随意挑选海外域名注册商（都支持免费 WHOIS 隐私保护、不需要国内实名，适合接 Cloudflare）：
1. Porkbun（https://porkbun.com）
2. Spaceship（https://www.spaceship.com）
3. Dynadot（https://www.dynadot.com）
4. Cloudflare Registrar（https://www.cloudflare.com/products/registrar/）
5. Namecheap（https://www.namecheap.com）

> 为什么一定要自己买？ `sslip.io`、`nip.io` 这类免费域名，没法把 DNS 交给 Cloudflare 管理，也就开不了小黄云。
- 不用买它附加的 SSL 证书、主机之类的东西。

---

## 第 2 步：把域名交给 Cloudflare 管理

1. 登录 [Cloudflare](https://dash.cloudflare.com/)（免费账号就够）→ 添加域名（Add a site） → 输入你的域名 → 套餐选 Free。
2. Cloudflare 会给你两个 Nameserver 地址（形如 `xxx.ns.cloudflare.com`、`yyy.ns.cloudflare.com`）。
3. 回到你买域名的网站 → 找到这个域名的 Nameservers（名称服务器）设置 → 删掉原来的，把 Cloudflare 给的两条完整粘贴进去，保存。（如果是在 Cloudflare Registrar 买的，域名本来就在 Cloudflare，这一步跳过。）
4. 回到 Cloudflare，点 立即检查域名服务器（Check nameservers now），状态变成 Active，就完成。

---

## 第 3 步：添加 DNS 记录，打开小黄云

在 Cloudflare → DNS → Records 添加一条记录：

| 类型 | 名称 | 内容 | 代理状态 |
|---|---|---|---|
| A | `@`（代表域名本身） | 你的 VPS 的 IP | 打开（橙色小云朵） |

- 小黄云一定要打开。打开后，别人解析你的域名拿到的是 Cloudflare 的 IP，看不到你的真实 IP，国内流量也就绕开了被墙的那个 IP。
- 灰色小云朵 = 只做 DNS 解析，流量还是直连你的 IP，等于没救。
- 额外的好处：以后换服务器，只要在这里把 IP 改成新机器的，所有客户端都不用改。

---

## 第 4 步：选 SSL 模式（容易出错的一步）

Cloudflare → SSL/TLS → 概述，有几种模式：

| 模式 | 访客 → Cloudflare | Cloudflare → 你的服务器 | 你的服务器要做什么 |
|---|---|---|---|
| Flexible（灵活） | HTTPS | HTTP（80 端口，明文） | 什么证书都不用，开个 80 端口就行 |
| Full | HTTPS | HTTPS | 要有证书（自签的也行） |
| Full (Strict) | HTTPS | HTTPS | 要有受信任的证书（比如 Cloudflare 的源服务器证书） |

### 怎么选

- 有节点：必须用 Full (Strict)。VLESS 协议本身不加密，用 Flexible 的话，Cloudflare 到你服务器这一段就是明文，你用节点访问的普通 HTTP 网站在这一段会被看光。
- 只有网站：推荐也用 Full (Strict)。图省事可以用 Flexible，代价是 Cloudflare 到你服务器这一段是明文。

> 我们自己当时用的是 Flexible，后来才意识到节点不该这么配。下面以 Full (Strict) 为主来写，Flexible 的做法放在 4.3。

### 4.1 生成源服务器证书（未实测）

Cloudflare 免费给你签一张证书，专门用在"Cloudflare → 你的服务器"这一段，有效期最长 15 年，不用续。

1. Cloudflare → SSL/TLS → 源服务器（Origin Server）→ 创建证书。
2. 保持默认（让 Cloudflare 生成私钥，RSA），主机名里有 `你的域名` 和 `*.你的域名`，有效期选 15 年 → 创建。
3. 格式选 PEM。把"源证书"那一大段复制下来，存成 `origin.pem`；把"私钥"那一段存成 `origin.key`。
4. ⚠️ 私钥只显示这一次，关掉页面就再也看不到了，一定先存好。
5. 把这两个文件传到服务器上，比如放在 `/etc/ssl/cf/`。私钥权限设成只有自己能读：`chmod 600 /etc/ssl/cf/origin.key`。

### 4.2 打开 Full (Strict)

Cloudflare → SSL/TLS → 概述 → 选 Full (Strict)。

服务器上要先按第 5 步（网站）或第 7 步（节点）把证书用上，再切这个模式；不然切完马上会报 525。

### 4.3 省事方案：Flexible（只给网站用，不要用在节点上）

⚠️ Flexible 模式只对 443 端口生效。8443、2053、2083、2087、2096 这些 HTTPS 端口会自动退回 Full 模式，也就是 Cloudflare 会用 HTTPS 去连你的服务器。服务器那个端口要是只跑明文，就会报 525。所以用 Flexible 时，所有 HTTPS 访问都走 443，用路径来区分不同的服务。

⚠️ 选 Flexible 之后，一定要做两件事，不然会无限重定向：
1. Cloudflare → SSL/TLS → 边缘证书 → 关闭 Always Use HTTPS（始终使用 HTTPS）。
2. 服务器上的反向代理不要再自己跳转到 HTTPS（Caddy 的做法见第 5 步）。

---

## 第 5 步：服务器这边配 Caddy（只有网站的、两样都有的看）

用路径来区分不同的服务，所有东西都从一个域名、一个端口进来。

### 5.1 Full (Strict) 的写法（推荐，未实测）

```caddyfile
你的域名:443 {
    tls /etc/ssl/cf/origin.pem /etc/ssl/cf/origin.key    # 第 4.1 步生成的源服务器证书

    encode gzip

    handle_path /api/* {
        reverse_proxy api:8000
    }

    handle_path /chat/* {
        reverse_proxy 172.18.0.1:8788 {
            flush_interval -1      # 流式回复（AI 打字效果）要加这一行，必须写在 reverse_proxy 的花括号里面
        }
    }

    handle {
        respond "Not Found" 404
    }
}
```

- Caddy 在 Docker 里的话，`docker-compose.yml` 里要把 443 端口映射出来（`"443:443"`），并把证书目录挂进容器（比如 `/etc/ssl/cf:/etc/ssl/cf:ro`）。
- 写了 `tls 证书 私钥`，Caddy 就不会再去自动申请证书。

### 5.2 Flexible 的写法（我们当时用的）

```caddyfile
{
    auto_https off
}

:80 {
    encode gzip
    # 下面的 handle_path 和 5.1 里一样
}
```

`auto_https off` 是为了不让 Caddy 把 HTTP 自动跳转到 HTTPS，否则会和 Cloudflare 来回踢，变成无限重定向。

### 5.3 这些数字，哪些要照抄

| 数字 | 出现在哪 | 能不能改 |
|---|---|---|
| `443` | 5.1 里 Caddy 监听的端口，第 7 步客户端的端口 | 照抄 |
| `80` | 5.2 里 Caddy 监听的端口 | 照抄。Flexible 模式下 Cloudflare 回源固定连 80 |
| `10000` | 第 7 步，xray 在服务器内部的端口 | 随便填，Caddy 里和 3x-ui 里填同一个就行 |
| `8000`、`8788` | Caddyfile 示例 | 我们自己服务的端口，换成你的 |
| `172.18.0.1` | Caddyfile 示例 | Docker 网络里宿主机的地址，每台机器可能不一样，用 `docker network inspect <你的网络名>` 看里面的 Gateway |
| `/api/`、`/chat/` | Caddyfile 示例 | 路径名自己起 |

### 5.4 让配置生效，再检查一遍

如果 Caddyfile 是以单个文件（`:ro`）挂载进容器的，千万别用 `sed -i` 原地改，直接先 `docker stop`，用 `cat >` 覆盖写入后再 `docker start`；不是只读挂载的话，改完直接重启容器最稳：
```bash
docker restart <caddy容器名>
```

再确认这几件事：
```bash
docker port <caddy容器名>          # 要能看到 443（Flexible 是 80）映射到了宿主机。输出是空的，说明端口根本没映射出来
curl -k --resolve 你的域名:443:127.0.0.1 https://你的域名/api/    # Full (Strict)：在服务器本机测试
curl -i http://127.0.0.1/api/      # Flexible：在服务器本机测试
ufw status                         # 443（Flexible 是 80）要放行
```
本机测试能拿到 200/404/502 都算正常；如果是 000，说明 Caddy 根本没在监听。

---

## 第 6 步：排查 521 / 525 / 526 / 重定向

排查顺序：先查服务器内部，再查 Cloudflare。源站本身没通的时候，在 Cloudflare 后台怎么改开关、改 DNS 都没用。

| 你看到的 | 真正的原因 | 怎么办 |
|---|---|---|
| 521 Web server is down | Cloudflare 连不上你服务器的端口（Full 系列是 443，Flexible 是 80） | 用 5.4 那几条命令查：端口有没有映射、本机能不能访问、防火墙有没有放行 |
| 525 SSL handshake failed | Cloudflare 用 HTTPS 去连，服务器握不上手。常见两种：① 选了 Full / Full (Strict)，服务器 443 上没有证书，或者 Caddy 没加载上证书；② 用了 8443、2053 这类非 443 端口，Flexible 不生效，自动退回 Full | 按第 4.1 步和 5.1 把证书用上；用 Flexible 的话只走 443 |
| 526 Invalid SSL certificate | 选了 Full (Strict)，服务器用的证书 Cloudflare 不认：自签的、过期的、或者和域名对不上 | 换成第 4.1 步的源服务器证书 |
| 重定向次数过多 | Flexible 模式 + 服务器自己又跳转到 HTTPS，两边来回踢 | 关掉 Always Use HTTPS，Caddy 加上 `auto_https off` |

到这一步，网站和 API 已经能通过 `https://你的域名/路径` 访问了。

---

## 第 7 步：把节点也接回来

原来的 Reality 节点是直连 IP 的，IP 被墙以后就废了，要换成能走 Cloudflare 的协议：VLESS + WebSocket + TLS，走 443。

> 为什么不用 2052 这类明文端口：VLESS 本身不加密，明文过 Cloudflare 等于裸奔；而且新版 Xray 客户端会直接拒绝没加密的 VLESS，一连就退出。

两种情况选一种：

### 7.1 服务器上有 Caddy（和网站共用 443）

节点躲在网站后面，用一个只有你知道的秘密路径区分开。

1. 在 3x-ui 里添加入站：协议 `vless`，端口随便填一个（比如 `10000`），传输 `ws`，路径填 `/一串随机字符`，TLS 选"无"（TLS 由 Caddy 负责）。
2. ⚠️ 入站里至少要有一个客户端（会自动生成 UUID），否则导出的链接和二维码是空的。
3. 在 Caddyfile 的站点里加一段，把这个路径转给 xray：
   ```caddyfile
   handle /一串随机字符* {
       reverse_proxy 172.18.0.1:10000
   }
   ```
   这里用 `handle` 不用 `handle_path`，路径原样转过去，所以客户端、Caddy、3x-ui 三个地方填的是同一个路径，不容易对不上。
4. 这个端口不用在防火墙里开放，它只在服务器内部用。

### 7.2 服务器上只有节点（没有 Caddy）（未实测）

让 3x-ui 自己用源服务器证书做 TLS。

1. 在 3x-ui 里添加入站：协议 `vless`，端口 `443`，传输 `ws`，路径 `/一串随机字符`。
2. 安全（Security）选 TLS，证书选"文件路径"，填第 4.1 步存的 `/etc/ssl/cf/origin.pem` 和 `/etc/ssl/cf/origin.key`，SNI 填你的域名。
3. ⚠️ 入站里至少要有一个客户端。
4. 防火墙放行 443：`ufw allow 443/tcp`。

### 客户端怎么填（两种情况一样）

- 地址：先填你的域名，测通以后再换成 Cloudflare 优选 IP（见第 8 步）
- 端口：443
- 传输：ws，路径填那个秘密路径
- 伪装域名（ws host）和 SNI：都填你的域名。Cloudflare 就是靠它判断这条流量该转给谁的，不能填成 IP
- TLS：打开

这样节点流量看起来就是普通的 HTTPS 网页访问。原来的 Reality 入站可以删了，顺便把防火墙里不再用的端口也关掉。

> 想多留一条备用线路，直接继续走 443 端口 + 不同的秘密路径即可。

---

## 第 8 步：挑一个你使用的运营商通得好的 Cloudflare IP

客户端的"地址"一栏填的 Cloudflare IP，不同运营商、不同地区通不通差别很大。
- 在自己电脑上测（最准，测的就是你自己的网络）：用 [CloudflareSpeedTest](https://github.com/XIU2/CloudflareSpeedTest)，下载后直接运行，它会自动测一批 Cloudflare IP 的延迟和速度，把最快的几个列出来。测的时候先关掉代理。
- 在网页上测全国各地：[itdog TCPing](https://www.itdog.cn/tcping/) 填 `某个Cloudflare IP:443`，能看到各省、各运营商通不通、延迟多少。适合帮别人挑，或者对比几个候选 IP。
- 挑出来的 IP 填进客户端的"地址"，伪装域名和 SNI 仍然填你的域名。过一段时间变慢了，就重新测一次。
- 测速显示 -1 不一定是不通。走 Cloudflare 的 TLS 节点首次握手要 3 秒左右，sing-box 的测速直接判成超时。以能不能打开网页为准，或者把 Core 改回 Xray 再测。

---

## 第 9 步：所有用到旧地址的地方都要改

IP 换成域名之后，所有填过 `http://IP:端口` 或旧域名的地方都要改一遍，否则就会出现"网站好了，但某个 App 还是连不上"。
例子：
- 节点客户端（电脑、手机）：导入新的分享链接
- 推送 App：删掉旧订阅，用新地址重新订阅
- 手机上的聊天客户端：接口地址改成 `https://你的域名/v1`
- 自定义连接器（如 MCP 服务）：地址改成 `https://你的域名/秘密路径/mcp`
- 服务器上程序自己的配置：比如推送里的跳转链接、对外公开的网址

---

## 整个流程一句话版

买域名 → 交给 Cloudflare 管理 → 添加 A 记录并打开小黄云 → 生成源服务器证书，SSL 选 Full (Strict) → 服务器上用这张证书听 443（网站用 Caddy，只有节点就让 3x-ui 自己来）→ 先在服务器本机测通再看 Cloudflare → 节点走 VLESS + WS + TLS + 443 + 秘密路径 → 挑优选 IP → 把所有旧地址都换掉。

---

## 附录：Cloudflare 52x 报错速查

| 错误码 | 意思 | 跟 521 怎么区分 | 先查什么 |
|---|---|---|---|
| 520 | 源站返回了 Cloudflare 看不懂的响应 | 连上了，但服务器回的是空响应，或者程序中途崩了 | 服务的日志（`docker logs <容器名>`），看后端是不是崩了 |
| 521 | 源站拒绝连接 | 端口没开，Cloudflare 一连就被拒 | 第 5.4 步那几条命令 |
| 522 | 连接源站超时 | 请求发出去了，半天没人应答（数据包被丢了） | 云厂商的安全组或防火墙有没有把端口挡掉；服务器是不是卡死或断网了 |
| 523 | 源站不可达 | 根本找不到去你服务器的路 | Cloudflare 里 A 记录的 IP 有没有填错；服务器是不是关机了 |
| 524 | 源站响应超时 | 连上了，但服务器超过约 100 秒都没开始回复 | 后端处理太慢，比如 AI 一直不出第一个字。让接口尽早开始流式输出，或者把慢任务改成后台执行 |
| 525 | SSL 握手失败 | Cloudflare 用 HTTPS 去连，服务器那边握不上手 | 服务器 443 上有没有用上源服务器证书（第 4.1、5.1 步）；用 Flexible 的话只走 443 |
| 526 | 源站证书无效 | 用的是 Full (Strict)，服务器证书是自签的、过期了，或者和域名对不上 | 换成 Cloudflare 的源服务器证书（第 4.1 步） |
