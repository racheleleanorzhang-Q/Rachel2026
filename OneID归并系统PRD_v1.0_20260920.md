# OneID归并系统PRD\_v1\.0\_20260920

## 一、背景与目标

### 1\.1 背景：问题本质

OneID 想解决的是**识别自然人全生命周期触点**，但现有数据里只有两类实体和它们之间的关系：

- **account\_id**（由 mobile 分配，代表"业务操作账户"）

- **id\_card**（在试驾/下单/交付各环节被登记，代表"某次业务行为所声明的自然人"）

**关键事实（已确认）：**

1. account\_id 与 id\_card 之间是**纯粹的多对多关系**

2. **没有任何场景（试驾/下单/交付）天然可信**——严肃场景（下单/交付）依然可能登记他人身份证（家庭代购车）

3. **没有独立的"手机-身份证联名实名认证"环节**——不存在能100%锚定这个 account\_id 本人是谁的强字段

4. 反过来，同一自然人也可能持有**多个 mobile → 多个 account\_id**（一人多号），这两个 account\_id 之间同样没有直接关联字段

**结论**：这是一个没有可信锚点的图归并问题，不是简单的关联字段打通问题。任何试图用图的连通性（谁跟谁有边）做自动归并的方法，都会因为家庭/朋友间的交叉代理行为，产生**传递闭包式误并**——把一个家庭甚至更大的社交圈错误合并成同一个人。这是方案设计必须首先规避的风险。

> 附表：[用户数据探查](https://li.feishu.cn/wiki/UeZZwmuoViWdO3kY0FKcVzvln87?sheet=914228)

#### account\_id ↔ id\_card 是多对多图，无边天然可信

现有数据中，account\_id（手机号锚定）与身份证 id\_card（试驾/下单/交付环节登记）之间不存在**可信**绑定关系，无法通过简单字段关联识别自然人。同时系统中，不同业务场景下引入了微信 UnionID、充电订单 user\_id、VIN 绑定身份证、设备指纹等多源信号，需要统一设计融合方案。

### 1\.2 问题拆解

现有断裂分两类，性质完全相反，不能用同一逻辑处理：

| 类型 | 现象 | 期望处理 | 难点 |
|---|---|---|---|
| **断裂1：一人多号** | 同一自然人注册了 mobile00a、mobile00d → account\_id\_1、account\_id\_2 两个账户 | 应该**合并**为一个 OneID | 两个 account\_id 之间没有直接关联字段，只能通过它们各自关联的 id\_card 间接推断，但这个推断本身不可信 |
| **断裂2：一号多人** | account\_id\_1 在试驾登记 card\_B（家人），下单登记 card\_C（另一家人） | **不应合并**成一个人，但要保留"这个账户服务过谁"的关系 | 场景严肃程度不能作为置信度判据，因为交付这种强场景一样会出现代他人操作的情况 |

两类断裂如果混用同一套"id\_card 相同就合并"的规则：

- 断裂1能解决一部分（如果两个账户都恰好登记过本人身份证）

- 但断裂2会被同一规则误伤——只要 account\_id\_1 和 account\_id\_3（属于家人B）都登记过 card\_B，就会被错误拉到一起，进而通过传递性把更多不相关账户连成一片

### 1\.3 目标

构建一套**多源 ID 置信度打分 + 关系图归并**系统，产出：

1. **Person 节点**：高置信度归并的自然人实体

2. **带标注关系图**：所有 ID 关联边 + 置信度分数，供下游业务方按需消费

### 1\.4 核心原则（不可违背）

- ❌ 不追求"一人一 ID"确定性唯一映射

- ❌ 场景类型（试驾/下单/交付）不作为置信度信号

- ❌ 禁止传递闭包式图连通合并（A-B-C 不等于 A=C）

- ✅ 合并操作与关系保留操作分离

- ✅ 置信度分层，下游业务方自选阈值

---

## 二、数据输入

### 2\.1 强锚点信号（参与打分，可触发合并）

| 信号 | 来源场景 | 来源权重 | 备注 |
|---|---|---|---|
| id\_card\_hash | VIN 车主绑定 | ×1.3 | 车辆登记，关联实际使用，可信度最高 |
| id\_card\_hash | 交付环节实名 | ×1.1 | |
| id\_card\_hash | 试驾/下单环节 | ×1.0（默认） | |
| 微信 UnionID | 微信授权登录 | ×1.0 | 一般一人一个，作为独立强锚点 |

### 2\.2 辅助信号（仅微调置信度，不独立触发合并）

| 信号 | 角色 | 处理方式 |
|---|---|---|
| 设备指纹 | 弱信号，家庭共享风险高 | 同向验证时微调置信度档位 |
| 微信 OpenID | UnionID 的子集 | 能追溯到 UnionID 时归并，否则忽略 |

### 2\.3 账户层内部别名（不参与打分，合并后直接挂载）

| 信号 | 处理方式 |
|---|---|
| charge\_acct\_id（充电订单 user\_id） | 与 account\_id 直接映射，Person 合并后挂载为"充电域触点 ID" |

### 2\.4 关键数据源表（已知）

- 账号表（account\_id ~ mobile）

- 三方绑定关系 user\_binding 表（account\_id ~ unionID 等三方绑定关系）

- 身份证绑定表 mall\_profile.certificate（accountID ~ id\_card）

- VIN 车主绑定表（account ~ vin）

- 微信授权登录表（UnionID/OpenID）

- 充电域映射表（charge\_acct\_id ~ account\_id）

---

## 三、核心算法设计

### 3\.1 单证据链置信度打分

对每条（account\_id，锚点值）边，分别计算 score\_idcard 和 score\_unionid，维度如下：

| 打分维度 | 含义 | 方向 |
|---|---|---|
| 频次 | 该 account 关联该锚点的次数 | 越高越可信 |
| 时间跨度 | 首次~最近一次出现间隔 | 跨度越长越可信 |
| 场景覆盖度 | 出现在几个不同业务环节 | 多环节 > 单环节 |
| 相对占比 | 该锚点在该账户所有关联锚点中的占比 | 避免高活跃账户所有值都虚高 |
| 反向热度校验 | 该锚点被多少其他 account 共享 | 共享越多越降权（家庭代理场景） |

输出分三档：**高（>0.8）/ 中（0.3~0.8）/ 低（<0.3）**

> PS：阈值为示例，需跑实际数据分布后校准

### 3\.2 双证据链交叉验证（合并判定）

| id\_card\_hash 链 | UnionID 链 | 合并结论 |
|---|---|---|
| 高 | 高（同一 UnionID） | **强合并**，最高置信度 |
| 高 | 无数据 / 中低 | **合并**，单锚点高置信度成立 |
| 中 | 高 | **合并**，UnionID 补强 |
| 高 | 高，但指向**不同** UnionID | ❌ **矛盾，转存疑队列，人工复核** |
| 中/低 | 中/低/无 | **不合并**，仅保留关系边 |

**一人多号合并条件（严格）：** account\_id\_1 和 account\_id\_2 必须同时对**同一张 id\_card** 打出"高"置信度，才通过该 id\_card 桥接合并。

### 3\.3 设备指纹微调规则

- 两账户已判定"合并" + 长期共享设备 → 提升至最高置信度档

- 两账户已判定"合并" + 从未共享设备 → 不否定结论，打"待观察"标记

---

## 四、产出物定义

### 4\.1 Person-ID 映射宽表（开发用，直接 JOIN）

每个 account\_id 单独一行，便于开发直接关联查询：

| 字段 | 类型 | 说明 |
|---|---|---|
| person\_id | string | 自然人唯一 ID |
| account\_id | string | 操作账户 ID |
| union\_id | string | 微信 UnionID（可空） |
| open\_id | string | 微信 OpenID（可空） |
| vin | string | 关联车辆 VIN（可空，可多行） |
| id\_card\_hash | string | 身份证 hash（可空） |
| charge\_user\_id | string | 充电域用户 ID（可空） |
| confidence\_level | enum | 强合并 / 合并 |
| updated\_at | timestamp | 最近更新时间 |

**示例数据：**

| person\_id | account\_id | union\_id | open\_id | vin | id\_card\_hash | charge\_user\_id | confidence\_level |
|---|---|---|---|---|---|---|---|
| person\_00001 | acc\_001 | unionid\_wx\_A001 | openid\_wx\_001 | VIN001XYZ | hash\_abc123 | chg\_201 | 强合并 |
| person\_00001 | acc\_007 | unionid\_wx\_A001 | openid\_wx\_007 | null | hash\_abc123 | chg\_388 | 强合并 |
| person\_00002 | acc\_042 | null | null | VIN042ABC | hash\_xyz789 | chg\_099 | 合并 |

> 说明：同一个 person\_id 可对应多行（一人多账户）；vin 字段若一人名下多辆车则展开多行。

### 4\.2 Person 节点聚合表（画像/本体系统用）

| 字段 | 类型 | 说明 |
|---|---|---|
| Person\_id | string | 系统生成的自然人唯一 ID |
| confidence\_level | enum | 合并置信度档位（强合并 / 合并） |
| anchor\_id\_card\_hash | array | 桥接用的 id\_card\_hash（可多个） |
| anchor\_unionid | string | 桥接用的 UnionID（可空） |
| account\_ids | array | 归并的 account\_id 列表 |
| charge\_acct\_ids | array | 挂载的充电域 ID 列表 |
| created\_at | timestamp | 创建时间 |
| update\_at | timestamp | 最近更新时间 |

### 4\.3 家庭单元实体表（中低置信度边建模输出）

中低置信度边不参与 Person 合并，但建模为正式"家庭单元"实体输出：

| 字段 | 类型 | 说明 |
|---|---|---|
| family\_id | string | 家庭单元唯一 ID |
| anchor\_id\_card\_hash | string | 聚合核心（共享的 id\_card\_hash） |
| account\_ids | array | 关联的 account\_id 列表（含置信度标注） |
| person\_ids | array | 已归并的 Person 节点 ID（可空） |
| member\_count | int | 关联账户数 |
| max\_score | float | 家庭内最高置信度边分数 |
| created\_at | timestamp | 创建时间 |
| updated\_at | timestamp | 最近更新时间 |

> 下游营销使用：以 family\_id 为单位进行家庭决策圈联合营销，业务方自担误召回风险。

### 4\.4 关系边表（全量保留）

| 字段 | 类型 | 说明 |
|---|---|---|
| account\_id | string | 操作账户 |
| anchor\_type | enum | id\_card\_hash / unionid |
| anchor\_value | string | 锚点值（已脱敏） |
| score | float | 0~1 置信度分数 |
| confidence\_level | enum | 高/中/低 |
| freq | int | 关联频次 |
| time\_span\_days | int | 时间跨度（天） |
| scene\_coverage | array | 出现的业务环节列表 |
| reverse\_heat | int | 反向热度（该锚点被共享的账户数） |
| source\_weight | float | 来源权重（VIN×1.3 / 交付×1.1 / 默认×1.0） |
| is\_merged | bool | 是否参与了 Person 合并 |

### 4\.5 存疑队列表

| 字段 | 类型 | 说明 |
|---|---|---|
| account\_id\_1 | string | 矛盾的账户 1 |
| account\_id\_2 | string | 矛盾的账户 2 |
| shared\_id\_card\_hash | string | 共享的身份证 hash |
| unionid\_1 | string | 账户 1 绑定的 UnionID |
| unionid\_2 | string | 账户 2 绑定的 UnionID |
| status | enum | 待复核 / 已处理 |
| review\_note | string | 人工复核备注 |

---

## 五、下游消费接口

业务方根据需求选择置信度阈值：

| 业务场景 | 使用方式 |
|---|---|
| 精准触达本人 | 只消费 Person 节点表（高置信度合并结果） |
| 家庭决策圈营销 | 使用关系边表中中低置信度边扩展范围（业务方自担误召回风险） |
| 家庭/社交圈分析 | 以 id\_card\_hash 为中心聚合关联 account\_id 集合 |

---

## 六、数据质量监控

| 监控指标 | 说明 |
|---|---|
| Person 节点覆盖率 | 有 person\_id 的 account\_id 占比 |
| 高置信度边占比 | 关系边表中高置信度边的比例 |
| 存疑队列规模 | 矛盾数据量，用于评估是否需要人工复核流程 |
| VIN 孤儿数据率 | 未关联 account\_id 的 VIN 记录占比 |
| charge\_acct\_id 挂载率 | 成功挂载到 Person 的充电 ID 占比 |

---

## 七、待决策

**Person 节点的回滚/更新机制**

- 新证据进来后，已合并的 Person 节点怎么办？重新计算？版本快照？

- 👉 需要：确认是"持续迭代覆盖"还是"带版本号的快照"模式

**数据质量监控的告警阈值和负责人**

- 监控指标有了，但触发告警的阈值是多少？报警给谁？

- 👉 需要：运营/数据团队确认
