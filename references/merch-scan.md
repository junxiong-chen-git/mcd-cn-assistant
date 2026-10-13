# 门店周边盘点细则

目标:给定关键词/区域,列出各门店「在售周边商品」清单与品类数。

## 事实与口径

- 门店菜单(`query-meals`,orderType=1, beType=1)只给「在售商品」,字段为 名称/编码/现价/分类/标签,**没有数量、没有库存**。
- 周边商品分两套:
  - **门店菜单里的周边**:和餐品一样挂在菜单(如「薯条脆卜卜毛绒周边」¥29.9),按门店不同而不同。
  - **麦麦商城周边**:`mall-points-products`(catRuleIds=`2>8`,shopId=2)以积分/现金兑换的实物周边,是**全局商城维度,不按门店**;只有状态(1仓库 2上架 3售罄 4下架 5预热),无数量。
- 唯一带余量数字的是主题活动场次:`query-partystore-session` 的 `leftNum`(是场次余位,不是商品库存)。
- 因此「各门店周边数量」只能理解为**在售周边品类数**,不是库存件数。回复必须写明这一点。

## 关键词表

`周边 毛绒 玩具 玩偶 挂件 徽章 礼盒 盲盒 手办 抱枕 马克杯 联名 公仔`

排除:餐品/饮料名;以及「玉米杯 / 中杯 / 大杯」这类规格用字(误命中)。

## 推荐流程(避免上下文被菜单撑爆)

1. 主任务先拿门店清单:`query-nearby-stores`(`searchType=2` + `city`+`keyword`;或 `searchType=1` 收藏门店)。记下 storeCode + 名称。
2. **一店一个采集子任务**(`sessions_spawn`,`collect=true`,`groupId` 分组,`outputSchema` 固定小输出,`context="isolated"`):子任务只调**一次** `query-meals`(storeCode / orderType=1 / beType=1),抽取关键词命中项,只返回 `{storeCode, storeName, merch[{name,code,price}], merchCount}`;禁止回传原始菜单。
3. 主任务用 `agents_wait` 收集全部 runId,合并成表。
4. 子任务的 `query-meals` 单次响应约 30–100 KB;若子任务工具集未挂载 `mcd-mcp__*`,它会改用原生 HTTP 调 MCP —— 结果同样有效,但会产生临时文件,仅登记不删。

## 输出模板

| 门店 | 编号 | 在售周边 | 品类数 |
| --- | --- | --- | --- |

合计:门店总数 / 有周边的门店数 / 周边品类总数。

## 备注

- 若要含「商城周边」,另调 `mall-points-products`(catRuleIds=`2>8`)并单列,不要与门店表混在一起。
- 菜单是分时段(早餐/正餐)的,盘点结果带时段性,回复里注明采集时间。
