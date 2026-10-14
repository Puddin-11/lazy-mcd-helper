[MCP_INTEGRATION.md](https://github.com/user-attachments/files/33432641/MCP_INTEGRATION.md)
# MCP 接入说明

本文档说明「懒人点餐助手」所使用的麦当劳 MCP（`mcd-mcp`）能力、调用链路与业务价值。

## 一、接入准备

1. 打开 WorkBuddy，进入连接器管理页。
2. 点击【自定义连接器】→【配置 MCP】，填入 `mcp-config.example.json` 中的配置（替换 `${MCD_MCP_TOKEN}` 为真实 Token）。
3. 回到【自定义连接器】，将 `mcd-mcp` 启用。
4. 若工具不可用，Skill 会明确提示用户前往连接器启用，**不会编造价格、券信息或活动内容**。

## 二、MCP Server

| 项 | 值 |
| --- | --- |
| Server 名称 | `mcd-mcp` |
| 传输方式 | `streamablehttp` |
| 地址 | `https://mcp.mcd.cn` |
| 鉴权 | `Authorization: Bearer ${MCD_MCP_TOKEN}` |

## 三、Tool 清单与用途

### 门店定位

| Tool | 用途 | 关键参数 |
| --- | --- | --- |
| `query-nearby-stores` | 到店自取（beType=1）/ 得来速（beType=5）门店查询 | `searchType`、`city`、`keyword` |
| `delivery-query-addresses` | 获取配送地址 `addressId` | 无 |
| `delivery-query-stores` | 外送场景按地址查可配送门店 | `beType`(2/6)、`addressId` |

### 时间与优惠资产

| Tool | 用途 | 数据性质 |
| --- | --- | --- |
| `now-time-info` | 获取当前时间，判定券的时段可用性 | 时间基准 |
| `query-store-coupons` | 指定门店 + 订单类型下可核销的券 | **唯一可承诺可用的一层** |
| `query-my-coupons` | 卡包内的券 | 资产清单，不代表可用 |
| `available-coupons` | 尚未领取的券 | 可领取层 |
| `query-promotions` | 门店满减 / 满折规则 | 仅企业团餐（beType=6） |
| `campaign-calendar` | 当月营销活动日历 | 新品、联名周边、抽奖 |

### 餐品与计价

| Tool | 用途 |
| --- | --- |
| `query-meals` | 获取餐品列表、`productCode` 与标价 |
| `query-meal-detail` | 获取餐品组成与特调选项 |
| `list-nutrition-foods` | 获取能量、蛋白质、脂肪、碳水、钠、钙 |
| `calculate-price` | 带券试算真实到手价 |

## 四、调用流程

```
① 口味记忆（本地，非 MCP）
   taste_profile.py show  →  加载 ~/.workbuddy/mcd-taste-profile.json
        │
        ▼
② 天气感知（Open-Meteo，非 MCP）
   weather.py --city 上海  →  温度 / 体感 / 餐饮倾向 / 外送建议
        │
        ▼
③ 时间基准
   now-time-info  →  判定券的时段窗口
        │
        ▼
④ 门店定位
   query-nearby-stores ──或── delivery-query-addresses → delivery-query-stores
        │  取得 storeCode（得来速/外送还需 beCode）
        ▼
⑤ 优惠资产盘点（并行）
   query-store-coupons ┐
   query-my-coupons    ├─ 四路合并，避免漏券
   available-coupons   ┤
   campaign-calendar   ┘
        │
        ▼
⑥ 餐品与规格
   query-meals → query-meal-detail → list-nutrition-foods
        │  取得 productCode、价格、标签
        ▼
⑦ 构造 4-6 组候选组合
        │
        ▼
⑧ 真实计价
   calculate-price（携带 couponId）→ 每组实付金额
        │
        ▼
⑨ 组合评分排序（本地）
   combo_ranker.py  →  口味 40 + 天气 25 + 预算 20 + 优惠 15
        │
        ▼
⑩ 输出 2-3 组方案
        │
        ▼
⑪ 回写口味画像
   taste_profile.py record  →  越用越准
```

## 五、关键参数规则

### 场景与参数映射

| 业务场景 | beType | orderType | beCode |
| --- | --- | --- | --- |
| 到店自取 | 1 | 1 | **不传**（传了会报错） |
| 得来速（DT） | 5 | 1 | **必传**，来自 `query-nearby-stores` |
| 麦乐送 | 2 | 2 | **必传**，来自 `delivery-query-stores` |
| 企业团餐 | 6 | 2 | **必传**，来自 `delivery-query-stores` |

参数错配是最高频的调用失败原因：
- 到店场景传了 `beCode` → 报错
- 得来速 / 外送场景漏传 `beCode` → 报错
- `query-promotions` 在非团餐场景调用 → 不可用

### 其他约束

- `storeCode` 必须全部由数字组成。
- `reservationDate` 仅预约场景传递，格式 `yyyy-MM-dd HH:mm`。
- `calculate-price` 返回的价格字段**单位为分**，展示时必须除以 100 转为元。

## 六、非 MCP 的外部依赖

| 依赖 | 用途 | 是否需要 Key |
| --- | --- | --- |
| Open-Meteo Geocoding API | 城市名 → 经纬度 | 不需要 |
| Open-Meteo Forecast API | 实时温度、体感、天气代码、降水 | 不需要 |

两者均为免费公共接口，无网络时脚本自动降级（`--mock` 离线段或询问用户）。

## 七、业务价值

懒人点餐助手不是简单地把 MCP 工具串起来，而是让 MCP 提供的数据在四个决策维度上产生合力：

| 维度 | MCP 提供什么 | 产生什么价值 |
| --- | --- | --- |
| **知你** | `query-meals` 提供标准餐品编码 | 口味画像以稳定的 `productCode` 为锚点，跨门店、跨门店菜单变动仍可匹配 |
| **知天** | —（由 Open-Meteo 补充） | 天气成为独立评分维度，占 25 分，解决"热天推热汤"的常识性错误 |
| **知价** | `calculate-price` 真实试算 | 报价即收银台价格，杜绝"到店付不出来"的假低价 |
| **知券** | `query-store-coupons` 门店级校验 | 区分"拥有券"与"能用券"，避免用户到柜台才发现券不可用 |

**核心价值主张：** 把「今天吃什么」这个每天都要重新决策的问题，从"打开菜单 → 逐个比价 → 回忆上次吃了什么 → 纠结 10 分钟"，压缩成"说一句话 → 直接得到 2-3 组可下单组合"。

## 八、错误处理策略

| 情况 | 处理方式 |
| --- | --- |
| MCP 未连接 | 提示用户前往连接器启用，停止后续调用 |
| 门店未找到 | 提示用户更换关键词，不虚构门店信息 |
| 券校验失败 | 降级为"你拥有该券但当前渠道不可用"，建议切换渠道 |
| 价格试算失败 | 展示接口返回的错误，**不回退到口算估算** |
| 天气接口失败 | 降级为 `--mock` 离线段或询问用户，**不编造温度** |
| 画像文件损坏 | 返回空画像并重新冷启动，不中断服务 |

## 九、数据安全声明

Skill 全程只读调用查询类接口。用户口味画像仅存本机 `~/.workbuddy/`，**不上传、不联网、不落库到服务端**。所有 MCP 查询数据仅在本轮对话内使用，不下单、不缓存、不外传。下单动作需由用户在对话中明确确认后才触发。
