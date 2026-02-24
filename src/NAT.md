好的，收到！已经把设备名从 `Core-SW` 统一改为 `R-Center`，这样更符合你当前的环境。

以下是更新后的 **NAT 教材**（只改了设备名，内容不变）：

---

# 网络地址转换（NAT，Network Address Translation）
---
顾名思义，本章将告诉你如何让私有IP上网。

---
众所周知，IPv4地址已经不够用了。NAT就是为了解决这个问题而生的——它能让一群设备共享一个公网IP上网。

和之前一样，思科和华为/华三的命令体系完全不同。请参照下表：

---

## ⚠️ 第一步：确认接口角色

配置NAT之前，必须搞清楚哪个接口是"内网"，哪个接口是"外网"：

> 💡 **小贴士**：实在分不清的话，记住"私网侧是里面是inside，公网侧是外面是outside"就行。

---

## 进入接口配置

确认好角色后，进入接口指定NAT角色：

| 厂商 | 指定内网口 | 指定外网口 |
|:---|:---|:---|
| **思科** | `ip nat inside` | `ip nat outside` |
| **华为** | `nat inside` | `nat outside` |
| **华三** | `nat inside` | `nat outside` |

---

## NAT的三种实现方式

### 1. 静态NAT（Static NAT）
> 一个内网IP对应一个固定公网IP，常用于对外发布服务器

| 厂商 | 配置命令 |
|:---|:---|
| **思科** | `ip nat inside source static [内网IP] [公网IP]` |
| **华为** | `nat static global [公网IP] inside [内网IP]` |
| **华三** | `nat static global [公网IP] inside [内网IP]` |

### 2. 动态NAT（Dynamic NAT）
> 内网IP从公网IP池里临时借一个用，用完归还，不常用

| 厂商 | 配置步骤 |
|:---|:---|
| **思科** | 1. `ip nat pool [池名] [起始IP] [结束IP] netmask [掩码]`<br>2. `access-list [号] permit [内网网段] [反掩码]`<br>3. `ip nat inside source list [号] pool [池名]` |
| **华为/华三** | 1. `nat address-group [组号] [起始IP] [结束IP]`<br>2. `acl [号] permit [内网网段] [通配符]`<br>3. `nat outbound [号] address-group [组号]` |

### 3. PAT/NAPT（Port Address Translation）
> **最常用的上网方式**：所有内网IP共享一个公网IP，通过端口号区分不同连接

| 厂商 | 配置命令 |
|:---|:---|
| **思科** | `ip nat inside source list [号] interface [外网口] overload` |
| **华为** | `nat outbound [号] address-group [组号]`（不加组号就是出接口） |
| **华三** | `nat outbound [号]` |

> 🚀 **重点掌握**：家庭和企业宽带上网用的就是PAT，也叫"端口复用"或"多对一NAT"。

---

## ⚠️ 第二步：确认哪个流量需要NAT

需要告诉设备：**哪些内网IP允许上网**。通常用ACL（访问控制列表）来定义：

| 厂商 | 定义允许上网的流量 |
|:---|:---|
| **思科** | `access-list 1 permit 192.168.1.0 0.0.0.255` |
| **华为** | `acl 2000 rule permit source 192.168.1.0 0.0.0.255` |
| **华三** | `acl basic 2000 rule permit source 192.168.1.0 0.0.0.255` |

> ⚠️ **注意**：思科用的是**反掩码**（0.0.0.255），华为/华三用的也是反掩码（通配符），千万别写成255.255.255.0！

---

## 完整配置示例

### 场景：让内网 192.168.1.0/24 通过外网口 G0/1 上网

### 思科 (Cisco)
```bash
R-Center# configure terminal
R-Center(config)# interface GigabitEthernet0/0      # 内网口
R-Center(config-if)# ip nat inside
R-Center(config-if)# exit
R-Center(config)# interface GigabitEthernet0/1      # 外网口
R-Center(config-if)# ip nat outside
R-Center(config-if)# exit
R-Center(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R-Center(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
R-Center(config)# end
```

### 华为 (Huawei)
```bash
<R-Center> system-view
[R-Center] interface GigabitEthernet0/0/0           # 内网口
[R-Center-GigabitEthernet0/0/0] nat inside
[R-Center-GigabitEthernet0/0/0] quit
[R-Center] interface GigabitEthernet0/0/1           # 外网口
[R-Center-GigabitEthernet0/0/1] nat outside
[R-Center-GigabitEthernet0/0/1] quit
[R-Center] acl 2000
[R-Center-acl-basic-2000] rule permit source 192.168.1.0 0.0.0.255
[R-Center-acl-basic-2000] quit
[R-Center] nat outbound 2000
[R-Center] return
```

### 华三 (H3C)
```bash
<R-Center> system-view
[R-Center] interface GigabitEthernet1/0/1           # 内网口
[R-Center-GigabitEthernet1/0/1] nat inside
[R-Center-GigabitEthernet1/0/1] quit
[R-Center] interface GigabitEthernet1/0/0           # 外网口
[R-Center-GigabitEthernet1/0/0] nat outside
[R-Center-GigabitEthernet1/0/0] quit
[R-Center] acl basic 2000
[R-Center-acl-ipv4-basic-2000] rule permit source 192.168.1.0 0.0.0.255
[R-Center-acl-ipv4-basic-2000] quit
[R-Center] nat outbound 2000
[R-Center] return
```

---

## 🔍 验证NAT状态

配置完了怎么确认生效了？

| 厂商 | 查看NAT转换表 | 查看NAT统计 |
|:---|:---|:---|
| **思科** | `show ip nat translations` | `show ip nat statistics` |
| **华为** | `display nat session all` | `display nat outbound` |
| **华三** | `display nat session` | `display nat outbound` |

### 思科输出示例
```bash
R-Center# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.5:12345  192.168.1.10:12345 8.8.8.8:80         8.8.8.8:80
```
> 看到类似输出说明内网192.168.1.10正在通过公网IP 203.0.113.5访问8.8.8.8的80端口。

---

## 常见问题

<details>
<summary>❓ 配置了NAT但上不了网？点我排查</summary>

1. **接口角色反了**：检查inside/outside是否标对
2. **ACL写错了**：确认反掩码是否正确（是0.0.0.255不是255.255.255.0）
3. **路由缺失**：设备需要有默认路由指向下一跳
   - 思科：`ip route 0.0.0.0 0.0.0.0 [下一跳IP]`
   - 华为/华三：`ip route-static 0.0.0.0 0 [下一跳IP]`
4. **外网口没IP**：或者IP不是公网IP（如果是运营商分配的私网IP，属于运营商级NAT，与本设备无关）
</details>

<details>
<summary>❓ PAT和动态NAT有什么区别？</summary>

| 类型 | 公网IP占用 | 最大连接数 | 适用场景 |
|:---|:---|:---|:---|
| **动态NAT** | 1个内网IP占用1个公网IP | 受公网IP数量限制 | 内网设备少，公网IP多 |
| **PAT** | 所有内网IP共用1个公网IP | 约65535个/每个公网IP | 家庭、企业宽带上网 |

**结论**：PAT是现在的主流，动态NAT基本被淘汰了。
</details>

---

## 总结

| 操作 | 思科 | 华为/华三 |
|:---|:---|:---|
| 标内网口 | `ip nat inside` | `nat inside` |
| 标外网口 | `ip nat outside` | `nat outside` |
| 定义内网范围 | `access-list [号] permit [网段] [反掩码]` | `acl [号] rule permit source [网段] [反掩码]` |
| 开启PAT | `ip nat inside source list [号] interface [外网口] overload` | `nat outbound [号]` |
| 查看转换表 | `show ip nat translations` | `display nat session all` |

---

## 一句话总结

**NAT就是让一群私网IP共用少数公网IP上网的技术，PAT是它最常用的形式。**
