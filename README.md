[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/arloor/rust_http_proxy)

基于 `hyper` 、 `axum` 、 `rustls` 的**静态资源托管服务器**、**正向代理**、**反向代理**、**API server**。

## 功能特性

1. 使用tls来对正向代理流量进行加密（`--over-tls`）。
2. 类Nginx的静态资源托管。支持gzip压缩。支持Accept-Ranges以支持断点续传（备注：暂不支持多range，例如 `Range: bytes=0-100,100-` ）
3. 支持反向代理（ `--reverse-proxy-config-file` ）。
4. 基于Prometheus的可观测，可以监控代理的流量、外链访问等。
5. 采集网卡上行流量，展示在 `/net` 路径下（读取 `/proc/net/dev` 或基于 `ebpf socket filter` ）
5. 支持多端口，多用户。
6. 每天定时加载tls证书，acme证书过期重新签发时不需要重启服务。
7. 连接空闲（10分钟没有IO）自动关闭。

提及的参数详见[命令行参数](#命令行参数)

## 安装说明

### linux amd64 可执行文件

```shell
curl -SLf https://us.arloor.dev/https://github.com/arloor/rust_http_proxy/releases/download/latest/rust_http_proxy -o /tmp/rust_http_proxy
install /tmp/rust_http_proxy /usr/bin/rust_http_proxy
/usr/bin/rust_http_proxy -p 7788
```

### Docker 安装 

> 通过Github Action自动更新release，永远是最新版，可放心使用

```shell
docker run --rm -it --net host --pid host docker.io/arloor/rust_http_proxy -p 7788
```

### ebpf版本安装

```bash
curl -SLf https://us.arloor.dev/https://github.com/arloor/rust_http_proxy/releases/download/latest/rust_http_proxy_bpf_static -o /tmp/rust_http_proxy
install /tmp/rust_http_proxy /usr/bin/rust_http_proxy
/usr/bin/rust_http_proxy -p 7788
```

或者

```bash
docker run --rm -it --privileged --net host --pid host docker.io/arloor/rust_http_proxy:bpf_static -p 7788
```

## 命令行参数

```shell
$ rust_http_proxy --help
A HTTP proxy server based on Hyper and Rustls, which features TLS proxy and static file serving

Usage: rust_http_proxy [OPTIONS]

Options:
      --log-dir <LOG_DIR>
          [default: /tmp]
      --log-file <LOG_FILE>
          [default: proxy.log]
  -p, --port <PORT>
          可以多次指定来实现多端口
           [default: 3128]
  -c, --cert <CERT>
          [default: cert.pem]
  -k, --key <KEY>
          [default: privkey.pem]
  -u, --users <USER>
          默认为空，表示不鉴权。
          格式为 'username:password'
          可以多次指定来实现多用户
  -w, --web-content-path <WEB_CONTENT_PATH>
          [default: /usr/share/nginx/html]
  -r, --referer-keywords-to-self <REFERER>
          Http Referer请求头处理 
          1. 图片资源的防盗链：针对png/jpeg/jpg等文件的请求，要求Request的Referer header要么为空，要么包含配置的值
          2. 外链访问监控：如果Referer不包含配置的值，并且访问html资源时，Prometheus counter req_from_out++，用于外链访问监控
          可以多次指定，也可以不指定
      --never-ask-for-auth
          if enable, never send '407 Proxy Authentication Required' to client。
          当作为正向代理使用时建议开启，否则有被嗅探的风险。
      --prohibit-serving
          禁止所有静态文件托管，为了避免被嗅探
      --allow-serving-network <CIDR>
          允许访问静态文件托管的网段，格式为CIDR，例如: 192.168.1.0/24, 10.0.0.0/8
          可以多次指定来允许多个网段
          如设置了prohibit_serving，则此参数无效
          如未设置任何网段，且未设置prohibit_serving，则允许所有IP访问静态文件
  -o, --over-tls
          if enable, proxy server will listen on https
      --reverse-proxy-config-file <FILE_PATH>
          反向代理配置文件
      --enable-github-proxy
          是否开启github proxy
      --append-upstream-url <https://example.com>
          便捷反向代理配置
          例如：--append-upstream-url=https://cdnjs.cloudflare.com
          则访问 https://your_domain/https://cdnjs.cloudflare.com 会被代理到 https://cdnjs.cloudflare.com
  -h, --help
          Print help
```

### SSL配置

其中，tls证书(`--cert`)和pem格式的私钥(`--key`)可以通过openssl命令一键生成：

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout /usr/share/rust_http_proxy/privkey.pem -out /usr/share/rust_http_proxy/cert.pem -days 3650 -subj "/C=cn/ST=hl/L=sd/O=op/OU=as/CN=example.com"
```

如需签名证书，请购买tls证书或免费解决方案（acme.sh等）

测试TLS Proxy可以使用curl （7.52.0以上版本）:

```bash
curl  https://ip.im/info -U "username:password" -x https://localhost:7788  --proxy-insecure
```

### 反向代理配置

```toml
[[YOUR_DOMAIN]]
location = "/" # 默认为 /

[YOUR_DOMAIN.upstream]
url_base = "https://www.baidu.com"
version = "H1" # 可以填H1、H2、AUTO，默认为AUTO
authority_override = "api.example.com" # 可选，覆盖发送给上游服务器的Host头
```

> 如果 `YOUR_DOMAIN` 填 `default_host` 则对所有的域名生效。

#### upstream配置说明

- `url_base`: 上游服务器的基础URL
- `version`: HTTP版本，可选值为 `H1`、`H2`、`AUTO`，默认为 `AUTO`
- `authority_override`: 可选参数，用于覆盖发送给上游服务器的Host头。如果不设置，则会自动从`url_base`中提取Host

#### 例子1: Github Proxy

在github原始url前加上`https://YOUR_DOMAIN`，以便在国内访问raw.githubusercontent.com、github.com和gist.githubusercontent.com

启动参数中增加 `--enable-github-proxy`，相当于以下配置：

```toml
[[default_host]]
location = "/https://gist.githubusercontent.com"

[default_host.upstream]
url_base = "https://gist.githubusercontent.com"

[[default_host]]
location = "/https://gist.github.com"

[default_host.upstream]
url_base = "https://gist.github.com"

[[default_host]]
location = "/https://github.com"

[default_host.upstream]
url_base = "https://github.com"

[[default_host]]
location = "/https://objects.githubusercontent.com"

[default_host.upstream]
url_base = "https://objects.githubusercontent.com"

[[default_host]]
location = "/https://raw.githubusercontent.com"

[default_host.upstream]
url_base = "https://raw.githubusercontent.com"
```

#### 例子2： 反向代理https://cdnjs.cloudflare.com

启动参数中增加 `--append-upstream-url=https://cdnjs.cloudflare.com`，相当于以下配置：

```toml
[[default_host]]
location = "/https://cdnjs.cloudflare.com"

[default_host.upstream]
url_base = "https://cdnjs.cloudflare.com"
```

#### 例子3: 改写Github Models的url为openai api的url格式

```toml
[[default_host]]
location = "/v1/chat/completions"

[default_host.upstream]
url_base = "https://models.inference.ai.azure.com/chat/completions"
```

#### 例子4: Host头覆盖 - 当上游服务器需要特定的Host头时

```toml
[["api.example.com"]]
location = "/api/"

["api.example.com".upstream]
url_base = "http://internal-service:8080"
authority_override = "api.internal.com:8080"  # 覆盖Host头为api.internal.com

[["cdn.example.com"]]
location = "/assets/"

["cdn.example.com".upstream]
url_base = "https://storage.cloud.com"
authority_override = "myapp.storage.com"  # 覆盖Host头为myapp.storage.com
```

## 可观测

### Prometheus Exporter

提供了Prometheus的Exporter。如果设置了`--users`参数，则需要在header中设置authorization，否则会返回`401 UNAUTHORIZED`。

```text
# HELP req_from_out Number of HTTP requests received.
# TYPE req_from_out counter
req_from_out_total{referer="all",path="all"} 4
# HELP proxy_traffic num proxy_traffic.
# TYPE proxy_traffic counter
# EOF
```

可以使用[此Grafana大盘Template](https://grafana.com/grafana/dashboards/20185-rust-http-proxy/)来创建Grafana大盘，效果如下

![alt text](grafana-template1.png)
![alt text](grafana-template2.png)

### Linux运行时的网速监控

在linux运行时，会监控网卡网速，并展示在 `/net` 。

![](speed.png)

## 客户端

- Clash系列
  - [clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) 
  - [ClashMetaForAndroid](https://github.com/MetaCubeX/ClashMetaForAndroid)
  - [mihomo(clash-meta)](https://github.com/MetaCubeX/mihomo/tree/Meta) 
- 自研玩具
  - Rust：[sslocal(fork shadowsocks-rust)](https://github.com/arloor/shadowsocks-rust)
  - Golang：[forward](https://github.com/arloor/forward)
  - Java: [connect](https://github.com/arloor/connect)

## Cargo Features

### bpf

使用ebpf来统计网卡出流量，仅在 `x86_64-unknown-linux-gnu` 上测试过。激活方式:

```bash
cargo build --features bpf
```

需要安装 `libbpf-rs` 所需的依赖：

**ubuntu 22.04 安装：**

```bash
apt-get install -y libbpf-dev bpftool cmake zlib1g-dev libelf-dev pkg-config clang autoconf autopoint flex bison gawk make
```

**centos stream 9 安装：**

```bash
yum install -y libbpf zlib-devel elfutils-libelf-devel pkgconf-pkg-config clang bpftool cmake autoconf gettext flex bison gawk make
```

### jemalloc

拥有更高的并发分配能力和减少内存碎片，不过会buffer更多的内存，因此top中RES数值会有上升。激活方式：

```bash
cargo build --features jemalloc
```

### aws_lc_rs

`aws_lc_rs` 和 `ring` 是 `rustls` 的两个加密后端。本项目默认使用 `ring` 作为加密后端，也可选择[aws_lc_rs](https://crates.io/crates/aws-lc-rs)作为加密后端。`aws_lc_rs` 相比ring主要有两点优势:

1. 在[rustls的benchmark测试](https://github.com/aochagavia/rustls-bench-results)中，`aws_lc_rs` 的性能要优于 `ring` 。
2. 支持美国联邦政府针对加密提出的[fips要求](https://csrc.nist.gov/pubs/fips/140-2/upd2/final)。

不过，使用 `aws_lc_rs` 会增加一些编译难度，需要额外做以下操作：

| 依赖的包 | 是否必须 |安装方式 |
| --- | --- | --- |
| `cmake` | 必须 | `apt-get install cmake` |

激活方式：

```bash
cargo build --no-default-features --features aws_lc_rs
```

## 高匿实现

代理服务器收到的http请求有一些特征，如果代理服务器不能正确处理，则会暴露自己是一个代理。高匿代理就是能去除这些特征的代理。具体特征有三个：

- 代理服务器收到的request line中有完整url，即包含schema、host。而正常http请求的url只包含路径
- 代理服务器收到http header中有Proxy-Connection请求头，需要去掉
- 代理服务器收到http header中有Proxy-Authentication请求头，需要去掉

本代理能去除以上特征。下面是使用tcpdump测试的结果，分别展示代理服务器收到的http请求和nginx web服务器收到的http请求已验证去除以上特征。

代理服务器收到的消息：

![](traffic_at_proxy.png)

Nginx收到的消息：

![](traffic_at_nginx.png)

可以看到请求URL和`Proxy-Connection`都被正确处理了。


## 容器测试

```bash
cargo clean
cargo build -r --features bpf_vendored
podman build . -f Dockerfile.test -t test --net host
podman run --rm -it --privileged --net host --pid host test 
```
