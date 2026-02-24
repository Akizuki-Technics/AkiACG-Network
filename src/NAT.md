# 网络地址转换（NAT，Network Address Translation）
---
顾名思义，本章将告诉你如何让私有IP上网。

---
众所周知，IPv4地址已经不够用了。NAT就是为了解决这个问题而生的——它能让一群设备共享一个公网IP上网。

当你捣鼓你的路由器的时候一般上行都会连一台家用路由器。而家用路由器绝大多数都没法设置静态路由或者接收动态路由表。它们只是为NAT而生的，也只会NAT。怎么办？那我们也NAT。

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
> 其实写错了也没关系，会有提示告诉你的。

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
> 此段代码未经本人测试，但是有以下反馈：华为只需在出接口配置 nat outbound [ACL] 即可，内网口保持默认。
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

<details>
<summary>当你用思科配置NAT的时候，如果你要配置PAT一定要加Overload。想知道为什么？点我。</summary>
   
## 思科配置 PAT 不加 overload 的后果
---
顾名思义，本隐藏章节将告诉你忘记敲 overload 会发生什么。

---
很多新手配置 PAT 时，照着教程敲了 `ip nat inside source list 1 interface g0/1`，结果内网电脑死活上不了网——**问题就出在忘了加 `overload`**。

在思科设备上，**`overload` 这个关键字决定了 NAT 的行为到底是"多对一"还是"多对多"**。不加它，设备就不会启用端口复用，后果很严重。

---

## ⚠️ 不加 overload = 动态 NAT

先看两条命令的对比：

| 命令 | 类型 | 含义 |
|:---|:---|:---|
| `ip nat inside source list 1 interface g0/1 overload` | **PAT（端口地址转换）** | 多对一，所有内网IP共享一个公网IP |
| `ip nat inside source list 1 interface g0/1` | **动态 NAT** | 多对多，每个内网IP独占一个公网IP |

**不加 `overload`，你配置的就是动态 NAT，而不是 PAT**。

---

## 💥 不加 overload 的四大后果

### 1. 公网IP不够用，直接瘫痪
- **PAT模式**：1个公网IP可以带整个内网（理论支持约65535个并发连接）
- **动态NAT模式**：**每个内网IP独占一个公网IP**，有多少公网IP，就只能让多少台设备同时上网

**后果**：如果你只有一个公网IP（家庭宽带、小企业专线常见情况），那么**只有第1台设备能上网，第2台开始全部失败**。

### 2. 配置报错（如果你用的是接口）
如果你用的是 `interface` 这种写法（如 `interface g0/1`），不加 `overload` 甚至可能报错：
```
%Pool ... not defined. Use 'ip nat pool' command
```
因为动态NAT需要一个地址池，而你只给了一个接口，设备不知道该怎么分配。

### 3. 端口不复用，连接数受限
- **PAT**：通过端口号区分不同连接，1个公网IP + 65535个端口 = 海量并发
- **动态NAT**：只换IP不换端口，1个公网IP = 1个连接（实际上是被端口限制了）

**后果**：即使用多个公网IP，每个IP同时只能被一台设备的一个会话占用，效率极低。

### 4. 外网无法主动访问内网（这不是好事）
虽然不加 overload 也能隐藏内网（和PAT一样），但**这不是优点**——你想要的只是让内网上网，而不是限制并发数。

---

## 🧪 实验对比

### 正确配置（PAT，加 overload）
```bash
R-Center(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R-Center(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
R-Center(config)# interface GigabitEthernet0/0
R-Center(config-if)# ip nat inside
R-Center(config-if)# interface GigabitEthernet0/1
R-Center(config-if)# ip nat outside
```

**效果**：内网50台电脑都能上网，`show ip nat translations` 看到同一个公网IP后面跟着不同端口：
```
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.5:12345  192.168.1.10:12345 8.8.8.8:80         8.8.8.8:80
tcp 203.0.113.5:23456  192.168.1.11:23456 8.8.8.8:80         8.8.8.8:80
```

### 错误配置（动态NAT，忘加 overload）
```bash
R-Center(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R-Center(config)# ip nat inside source list 1 interface GigabitEthernet0/1
%Pool ... not defined. Use 'ip nat pool' command   # 报错
```

即使强行用地址池配置：
```bash
R-Center(config)# ip nat pool PUBLIC 203.0.113.5 203.0.113.5 netmask 255.255.255.248
R-Center(config)# ip nat inside source list 1 pool PUBLIC
```

**效果**：只有第1台访问外网的设备能成功，第2台开始全部超时。

---

## 🔍 怎么判断当前配置是PAT还是动态NAT？

| 命令 | PAT 输出特征 | 动态 NAT 输出特征 |
|:---|:---|:---|
| `show ip nat translations` | 同一个公网IP，**端口号不同** | 不同内网IP对应**不同公网IP** |
| `show ip nat statistics` | 显示 `Total translations: 很多` | 显示 `Total addresses: 很少` |

---

## ✅ 正确用法总结

| 场景 | 命令 | 说明 |
|:---|:---|:---|
| 只有一个公网IP | `ip nat inside source list 1 interface g0/1 overload` | **必须加 overload** |
| 有多个公网IP，想让内网共享 | `ip nat pool POOL 203.0.113.5 203.0.113.10 netmask 255.255.255.248`<br>`ip nat inside source list 1 pool POOL overload` | **也要加 overload**，否则变成动态NAT |
| 真的想要动态NAT（极少见） | 不加 overload，但要有足够多的公网IP | 每个内网IP独占一个公网IP |

---

**在思科设备上，不加 `overload` 的 NAT 就不是 PAT，而是动态 NAT——如果你只有一个公网IP，那只有第1台设备能上网。**

</details>
