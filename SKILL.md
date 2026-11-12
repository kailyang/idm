---
name: semiconductor-idm-network-map
description: "生成半导体 IDM、晶圆厂、封测厂与 IC 设计企业的网络部署图和交互全览图，默认交付单文件 HTML。用于规划或呈现集团多子公司网段、Fab IT/OT 分区、生产机台、AMHS/FMCS、办公/研发/车间 VDI、内网与外网 VDI 接入、DMZ/iDMZ、五层防火墙、VLAN/IP、访问策略、OOB/SOC，以及可点击任意流向、悬停查看五元组、逐跳播放的数据包旅程。也用于把用户提供的资产表、网段表、防火墙策略或既有拓扑转换为统一可校验的交互图；不适用于真实设备配置下发或合规认证结论。"
---

# 半导体 IDM 网络全览图生成

## 目标

用**一份配置生成一份完整 HTML**，同时包含：

1. 集团广域与子公司网段；
2. Fab1 IT / DMZ / iDMZ / OT 分区部署；
3. 九大工艺与生产支撑区、具体机台和微隔离；
4. 办公、研发、车间桌面池及内网/外网 VDI 双接入；
5. 可点击、可悬停、可逐跳播放的全览拓扑；
6. 防火墙策略矩阵、VLAN/IP 表与评审清单。

配置是唯一事实源。不要在 HTML 各处分别硬编码 IP、VLAN、策略和路径，避免同一设备前后矛盾。

## 快速开始

优先使用随附生成器，而不是每次重写页面：

```bash
python scripts/validate_config.py assets/example-config.json
python scripts/build_map.py assets/example-config.json --output 半导体IDM网络全览图.html
python scripts/smoke_test.py
```

路径均相对本技能目录。生成器只依赖 Python 标准库；输出为单文件、原生 JS、可离线打开的 HTML，字体加载失败不影响使用。

复制 `assets/example-config.json` 到用户任务目录后修改，**不要直接改技能内的示例文件**。字段说明见 `references/data-model.md`。

## 按需读取

- 需要确认 IT/OT 模型、SEMI 协议、机台通信、远程运维或安全边界：读 `references/domain-baseline.md`。
- 需要填写或迁移配置、规划站点与 VLAN、定义访问路径：读 `references/data-model.md`。
- 需要修改模板、SVG 布局、播放、点击/悬停或做浏览器验收：读 `references/interaction-impl.md`。

## 工作流

### 1. 明确范围与证据状态

先确定：

- 交付是规划示例、现状还原，还是两者对照；
- 集团包含哪些站点，主图聚焦哪个 Fab；
- 是否必须覆盖机台、AMHS、FMCS、VDI、OOB、SOC；
- 用户是否提供真实 IP、VLAN、设备和策略。

将 `meta.data_mode` 设为 `example`、`planned` 或 `verified`：

- `example`：全部地址、企业名和型号视为演示数据，页面持续显示“示例数据”；
- `planned`：规划建议，页面标注“待现场确认”；
- `verified`：仅在用户材料可逐项追溯时使用，并填写 `meta.source`。

缺少的信息用示例补齐时必须显式标注，不把推测写成现状，不声称“已通过等保/IEC/SEMI 认证”。

### 2. 建立统一地址与安全域

- 集团站点默认各分配一个 `/16`，站点内按业务划分 `/24`；检查站点和 VLAN 子网重叠。
- IT 采用核心—汇聚—接入；OT 参考 Purdue 层级与 IEC 62443 的 Zone/Conduit。
- 安全域至少覆盖外部、DMZ、IT 办公/VDI、企业服务器、iDMZ、MES/EAP、工艺机台、SOC、OOB。
- Purdue 层级是安全参考，不等于 VLAN 编号，也不替代实际风险评估。

### 3. 清点资产和链路

节点至少记录：名称、类型、所属域、管理/业务地址、角色；实施级图纸另补型号、接口、带宽、介质、HA、机柜和端口。

链路至少记录：源/目的、协议、端口、动作、加密、经过的防火墙、策略号与业务解释。链路五元组无法确认时写“待确认”，不要伪造精确端口号。

### 4. 覆盖半导体的必备细节

默认包含：

- 光刻、刻蚀、薄膜、扩散/注入、CMP、清洗、量测、AMHS、FMCS 九个区域；
- 机台固定 IP、EAP 与 HSMS/GEM、NTP、日志、Service 维护口与生产口分离；
- MES、EAP、RMS/YMS/SPC、Historian、补丁/AV 仓库、协议网关和厂商跳板；
- 办公池、研发池、车间池、VDI Broker、计算/存储、AD/DNS/DHCP；
- 内网 VDI 与外网 VDI 两种接入，不能只画外网；
- FW1 边界、FW2 内部、FW3 IT 侧、FW4 OT 侧、FW5 工艺区微隔离；
- 默认拒绝、IT/OT 不直连、工艺区横向隔离、机台不主动访问互联网、OOB 业务拒绝。

### 5. 定义交互旅程

配置两类对象：

- `accesses`：内网和外网入口的 edge id 前缀；
- `targets`：VDI 桌面、ERP、MES 数据、机台操作、OOB 拒绝等目标的 edge id 后缀。

完整路径 = `access.prefix + target.suffix`。每条组合路径必须：

1. edge id 均存在且不重复；
2. 前一跳的 `to` 等于后一跳的 `from`；
3. 第一跳起点与接入的 `start_node` 一致；
4. `deny` 只出现在最后一跳；
5. 内外网共享的后段只在汇合节点之后复用。

路径文案按“这一跳做什么检查、为什么能/不能通过”撰写，不只重复设备名。

### 6. 生成与校验

```bash
python scripts/validate_config.py <config.json> --strict
python scripts/build_map.py <config.json> --output <语义化文件名.html>
```

先修完全部 ERROR，再处理 WARN。`--strict` 将警告也视为失败，适合交付前检查。

校验器覆盖：必填字段、id 唯一性、CIDR/VLAN、网关归属、同站点子网重叠、节点 IP 归属、edge 端点、策略引用、路径连续性、拒绝终点、双接入和示例标识。

### 7. 浏览器验收

静态截图不能证明交互正确。至少在真实浏览器完成：

- 内网 VDI 与外网 VDI 各播放一次到终点，确认跳数、起点和经过边界不同；
- 上一步、下一步、暂停、继续、重置均有效；
- 点击一条正常链路和一条变暗链路，详情与五元组正确；
- 点击设备卡片，悬停 edge/node，tooltip 不越出视口；
- OOB 场景在拒绝跳终止，不显示“连接成功”；
- 策略筛选、网段搜索、站点/工艺区内容、键盘操作可用；
- 390px 移动视口无横向溢出，桌面无节点/标签重叠；
- 禁用网络后仍能打开并操作，控制台无 JavaScript 错误。

模板修改后运行 `python scripts/smoke_test.py`，再按 `references/interaction-impl.md` 的清单复核。

## 输出要求

- 默认只交付一个语义化命名的 `.html`，不要额外交付模板、JSON 或备份，除非用户要求。
- 页面默认包含：元信息、集团站点、安全域、交互拓扑、VDI 路径、机台详图、策略矩阵、地址表、评审清单和图例。
- 所有可见按钮必须有真实行为；动画可暂停/重置，并尊重 `prefers-reduced-motion`。
- 不用 emoji；图标使用内联 SVG。无脚本时仍保留配置摘要、地址表和策略表的可读静态内容或明确降级提示。
- 最终说明区分“已实现功能”“示例/待确认数据”“未覆盖项”。不要把逻辑规划图描述成施工布线图或合规证明。

## 修改技能本身

只修改本技能时：

1. 同步更新配置示例、schema 文档、模板与校验器，避免接口漂移；
2. 运行 `python scripts/smoke_test.py`；
3. 用示例配置实际生成 HTML 并做浏览器验收；
4. 运行技能创建器的 `quick_validate.py`；
5. 保持普通文件夹，不创建 README、CHANGELOG、zip 或 `.skill`。
