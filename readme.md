# nftables
`nftables` 是 Linux 系统中的一个防火墙框架，用于替代旧的 `iptables`。它提供了更强大的性能、更加灵活的规则管理，并简化了防火墙配置。具体作用如下：

1. **网络包过滤**：`nftables` 用于控制进出计算机的网络流量，允许或阻止特定的网络包。它可以根据源/目的 IP 地址、端口号、协议等规则进行过滤。

2. **网络地址转换 (NAT)**：可以使用 `nftables` 配置网络地址转换，如源地址转换 (SNAT) 或目标地址转换 (DNAT)，来改变网络包的源或目的地址。

3. **流量计量**：`nftables` 允许记录网络流量的计数信息，可以用于流量监控或流量限速。

4. **高效性能**：与 `iptables` 相比，`nftables` 提供了更高的性能和更低的内存消耗。它使用了一种基于虚拟机（Netfilter虚拟机）的技术，能够在高流量场景下处理更高效。

5. **规则集合与命名空间**：支持使用规则集来组织规则，并且可以将规则按命名空间进行划分，便于管理。

6. **IPv4 和 IPv6 支持**：`nftables` 同时支持 IPv4 和 IPv6，统一的接口使得配置更加简便。

7. **灵活的规则表达**：通过支持更复杂的数据结构和表达式（如集群操作、位掩码操作等），`nftables` 提供了比 `iptables` 更强大的规则定义能力。

总的来说，`nftables` 提供了一个统一的网络包处理框架，支持更复杂的防火墙和流量控制策略，取代了 `iptables` 的老旧结构，并使网络配置更加简洁和高效。

是的，使用 `nftables` 可以将电脑设置成网关，从而实现网络地址转换 (NAT)，使得内网的设备能够访问外网。通过配置 `nftables`，你可以将流量转发到合适的接口，并通过源地址转换 (SNAT) 和路由规则来实现网关功能。

以下是使用 `nftables` 将电脑设置为网关的基本步骤：

### 1. 启用 IP 转发
首先，你需要启用 IP 转发，使得电脑能够转发网络流量。

编辑 `/etc/sysctl.conf` 文件，确保以下行存在且未被注释：

```bash
net.ipv4.ip_forward = 1
```

然后应用配置：

```bash
sudo sysctl -p
```

### 2. 配置 `nftables`
你需要配置 `nftables` 来启用 NAT 和流量转发。

#### 2.1 设置基本规则集
创建或编辑 `/etc/nftables.conf` 文件，添加如下规则：

```bash
#!/usr/sbin/nft -f

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif lo accept
        ct state established,related accept
        ip saddr 192.168.1.0/24 accept  # 允许来自内网的流量（假设内网地址段是 192.168.1.0/24）
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        iifname "eth1" oifname "eth0" accept  # 允许流量从 eth1 转发到 eth0（假设 eth1 是内网，eth0 是外网）
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        oifname "eth0" masquerade  # 启用源地址转换（SNAT），使内网设备通过 eth0 上网
    }
}
```

#### 2.2 应用 `nftables` 配置
保存并应用 `nftables` 配置：

```bash
sudo nft -f /etc/nftables.conf
```

### 3. 配置路由
确保内网设备的默认网关设置为该电脑的 IP 地址。比如，假设网关电脑的 IP 地址是 `192.168.1.1`，则内网设备的默认网关应该配置为 `192.168.1.1`。

### 4. 验证配置
确保一切正常工作，内网设备应该能够通过你的电脑访问外网。你可以使用如下命令检查转发和 NAT 是否正常：

```bash
sudo nft list ruleset
```

通过这种配置，你的电脑就充当了网关的角色，内网设备能够通过它访问互联网。

这段 `nftables` 配置的作用是设置网络地址转换 (NAT)，特别是启用 **源地址转换（SNAT）**，使得内网设备通过网关电脑的外网接口（如 `eth0`）访问互联网。下面逐项解释这段配置：

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        oifname "eth0" masquerade  # 启用源地址转换（SNAT），使内网设备通过 eth0 上网
    }
}
```

### 1. **`table ip nat`**
- `table ip nat` 表示你正在定义一个用于处理 IPv4 地址转换的 `nat` 表。`nat` 表是专门用于地址转换的，包括源地址转换（SNAT）和目标地址转换（DNAT）等操作。
- `table ip` 表示该规则适用于 IPv4 流量。如果你需要同时处理 IPv6 流量，可以使用 `table ip6`。

### 2. **`chain postrouting`**
- `chain postrouting` 是一个名为 `postrouting` 的链，用于处理数据包离开防火墙时的操作。`postrouting` 主要用于修改即将离开防火墙的网络包，通常用于执行 NAT 操作（如源地址转换）。
- 也就是说，所有从防火墙流出的流量（经由 `oifname` 指定的接口）都会匹配到 `postrouting` 链。

### 3. **`type nat hook postrouting`**
- `type nat` 表示该链专门用于处理 NAT 操作。
- `hook postrouting` 指定这个链是在数据包即将离开防火墙时执行的。`postrouting` hook 允许你在包离开前修改它的源地址、目标地址等。

### 4. **`priority 100`**
- `priority 100` 是该链的优先级，数字越低优先级越高。在 `nftables` 中，你可以给不同的链设置优先级，决定规则的应用顺序。
- `priority 100` 在所有 NAT 链中相对较低，意味着它会在其他 NAT 操作后应用。

### 5. **`policy accept`**
- `policy accept` 设置该链的默认策略为“接受”，即如果没有匹配的规则，默认允许包通过。
- 这意味着如果包没有被其他规则匹配，则允许包继续转发。

### 6. **`oifname "eth0" masquerade`**
- **`oifname "eth0"`**：`oifname` 指定了包离开时的目标接口。`"eth0"` 是外网接口，表示包是要通过 `eth0` 发往外部网络。
- **`masquerade`**：`masquerade` 是一种特殊的源地址转换（SNAT）操作，通常用于将内网设备的私有 IP 地址转换为网关设备的公共 IP 地址。
  - 当内网设备发送数据包到外网时，`masquerade` 会将包的源 IP 地址改为网关设备的外网接口（`eth0`）的 IP 地址。这样，内网设备的私有地址就不会暴露给外网，所有内网设备的请求看起来是来自网关本身。

### 总结：
- **源地址转换 (SNAT)**：`masquerade` 会修改数据包的源 IP 地址，将其从内网设备的私有地址转换为网关设备（即防火墙机器）的外网 IP 地址（`eth0` 接口的 IP）。
- 这样，内网设备就可以共享网关的外网连接，通过 `eth0` 接口访问互联网，而外网看到的所有请求都来自网关的公共 IP，而不是内网设备的私有 IP。

### 典型应用场景：
这段配置通常用于：
- **网关或路由器设置**：将内网设备的流量通过网关机器的外网接口转发，并在转发时对内网设备的 IP 进行源地址转换，使外部看到的请求源自网关而不是内网设备。
- **网络共享**：例如，将一台电脑设置为家用网络的网关，使得多个设备能够共享该电脑的互联网连接。

要使用 `nftables` 将计算机设置为网关，需要配置内核转发并设置 NAT（网络地址转换）规则。这通常用于将网络流量从一个网络接口转发到另一个网络接口。以下是基本的步骤：

### 1. 启用 IP 转发

首先，确保系统允许 IP 转发。可以通过以下命令启用：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

要使其在重启后生效，请编辑 `/etc/sysctl.conf` 文件，添加或修改以下行：

```bash
net.ipv4.ip_forward = 1
```

然后应用配置：

```bash
sudo sysctl -p
```

### 2. 配置 `nftables` NAT 规则

假设你有两个网络接口：
- `eth0`：连接到外部网络（例如互联网）。
- `eth1`：连接到局域网（LAN）。

接下来，使用 `nftables` 配置 NAT 和流量转发。

#### 2.1. 创建表和链

首先，创建一个 `nftables` 表并添加适当的链来处理 NAT 和流量转发：

```bash
sudo nft add table inet filter
sudo nft add table inet nat
```

#### 2.2. 配置 NAT

为外部接口添加 MASQUERADE 规则，使流量可以通过 `eth0` 转发：

```bash
sudo nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule inet nat postrouting oifname "eth0" masquerade
```

#### 2.3. 配置转发链

允许流量在 `eth1` 和 `eth0` 之间转发：

```bash
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; }
sudo nft add chain inet filter forward { type filter hook forward priority 0 \; }
sudo nft add chain inet filter output { type filter hook output priority 0 \; }

sudo nft add rule inet filter forward iifname "eth1" oifname "eth0" accept
sudo nft add rule inet filter forward iifname "eth0" oifname "eth1" accept
```

### 3. 保存配置

使用以下命令保存 `nftables` 配置，以确保重启后仍然有效：

```bash
sudo nft list ruleset > /etc/nftables.conf
```

并确保在系统启动时加载该配置：

```bash
sudo systemctl enable nftables
```

### 4. 测试

确保你的 `eth1` 网络接口设备能正常通过 `eth0` 访问外部网络。你可以通过运行如下命令检查转发状态：

```bash
ping -I eth1 8.8.8.8
```

如果设置正确，应该能够从内网通过网关访问互联网。

这就是使用 `nftables` 配置计算机作为网关的基本步骤。如果有进一步的问题，可以继续询问。