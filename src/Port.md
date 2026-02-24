# 端口配置基础
---
顾名思义，本章将告诉你如何配置端口。

---
众所周知，思科和华为华三的命令体系是完全不同的。

要开始配置端口，首先需要进入全局配置模式/视图

三家命令有不同，参照下表：
你说得对，思科和华为/华三的命令体系确实差异很大。对于初学者来说，**最容易混淆的就是进入端口视图和配置端口的命令**。

下面我帮你整理了一个**三家命令对照表**，以及进入配置模式的完整流程：

---

## 进入全局配置模式

无论哪家，第一步都是先进入全局配置模式：

| 厂商 | 进入全局配置模式 | 提示符变化 |
|:---|:---|:---|
| **思科 (Cisco)** | `configure terminal` (一般常用 `conf t`) | `Switch#` → `Switch(config)#` |
| **华为 (Huawei)** | `system-view` (或 `sys`) | `<Switch>` → `[Switch]` |
| **华三 (H3C)** | `system-view` (或 `sys`) | `<Switch>` → `[Switch]` |

> 💡 **小贴士**：华为和华三的命令体系基本一致，都源自早期的Comware系统。

---

## 进入端口视图

进入全局模式后，接下来要进入具体的端口进行配置：

| 厂商 | 进入以太网端口 | 示例 | 提示符变化 |
|:---|:---|:---|:---|
| **思科** | `interface [端口号]` | `interface GigabitEthernet0/1` | `Switch(config)#` → `Switch(config-if)#` |
| **华为** | `interface [端口号]` | `interface GigabitEthernet0/0/1` | `[Switch]` → `[Switch-GigabitEthernet0/0/1]` |
| **华三** | `interface [端口号]` | `interface GigabitEthernet1/0/1` | `[Switch]` → `[Switch-GigabitEthernet1/0/1]` |

> ⚠️ **注意**：命令虽然都是 `interface`，但**端口编号规则完全不同**！
> 具体情况请自行查看端口号，实在无法确认建议show/display int看一下端口
> 更高级的搜索会在后文展示，有深度学习需要可以查看。

---

## 端口编号规则对照

这是最容易搞混的地方：

| 厂商 | 端口编号格式 | 含义 |
|:---|:---|:---|
| **思科 (IOS)** | `GigabitEthernet0/1` | 模块号0/端口号1（模块0通常是板载端口） |
| **华为** | `GigabitEthernet0/0/1` | 槽位号0/卡号0/端口号1 |
| **华三** | `GigabitEthernet1/0/1` | 槽位号1/卡号0/端口号1 |

**举例说明**：
- 思科的 `G0/1` = 华为的 `G0/0/1`，但 ≠ 华三的 `G1/0/1`（因为华三槽位从1开始）

---

## 常用端口配置命令对比

| 配置项 | 思科 (Cisco) | 华为/华三 (Huawei/H3C) |
|:---|:---|:---|
| **描述** | `description [文本]` | `description [文本]` |
| **二层口转三层口** | `no switchport` | `undo portswitch` |
| **三层口转二层口** | `switchport` | `portswitch` |
| **配置IP** | `ip address [IP] [掩码]` | `ip address [IP] [掩码]` |
| **启用端口** | `no shutdown` | `undo shutdown` |
| **关闭端口** | `shutdown` | `shutdown` |
| **配置Access VLAN** | `switchport access vlan [号]` | `port access vlan [号]` |
| **配置Trunk** | `switchport mode trunk` | `port link-type trunk` |
| **允许VLAN通过** | `switchport trunk allowed vlan [号]` | `port trunk permit vlan [号]` |

---

## 实战示例

### 思科 (Cisco) 配置端口G0/1
```bash
Core-SW# configure terminal
Core-SW(config)# interface GigabitEthernet0/1
Core-SW(config-if)# description Connection-to-ACcess
Core-SW(config-if)# no switchport          # 改为三层口
Core-SW(config-if)# ip address 192.168.1.1 255.255.255.0
Core-SW(config-if)# no shutdown
Core-SW(config-if)# end
```

### 华为 (Huawei) 配置端口G0/0/1
```bash
<Core-SW> system-view
[Core-SW] interface GigabitEthernet0/0/1
[Core-SW-GigabitEthernet0/0/1] description Connection-to-ACcess
[Core-SW-GigabitEthernet0/0/1] undo portswitch    # 改为三层口
[Core-SW-GigabitEthernet0/0/1] ip address 192.168.1.1 24
[Core-SW-GigabitEthernet0/0/1] undo shutdown
[Core-SW-GigabitEthernet0/0/1] quit
```

### 华三 (H3C) 配置端口G1/0/1
```bash
<Core-SW> system-view
[Core-SW] interface GigabitEthernet1/0/1
[Core-SW-GigabitEthernet1/0/1] description Connection-to-ACcess
[Core-SW-GigabitEthernet1/0/1] undo portswitch    # 改为三层口
[Core-SW-GigabitEthernet1/0/1] ip address 192.168.1.1 24
[Core-SW-GigabitEthernet1/0/1] undo shutdown
[Core-SW-GigabitEthernet1/0/1] quit
```

---

## 总结

| 操作 | 思科 | 华为/华三 |
|:---|:---|:---|
| 进全局 | `conf t` | `system-view` |
| 进端口 | `interface g0/1` | `interface g0/0/1` |
| 给IP | `ip address x.x.x.x x.x.x.x` | `ip address x.x.x.x 24` |
| 启用 | `no shutdown` | `undo shutdown` |
| 退出一级 | `exit` | `quit` |
| 退回用户 | `end` | `return` |
