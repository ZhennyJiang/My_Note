# 购物车 + 子母订单 前端 API 契约设计

> 日期：2026-04-17
> 基础路径：`/api/frontend`
> 模块：broker-business-frontend-orch
> 对齐基线：2026-04-13-cart-parent-order-frontend-api.md

## 错误码定义

### 购物车错误码（1-102-006-xxx）

| 错误码        | 名称                         | HTTP 状态 | 描述                                     | 处理建议           |
| ------------- | ---------------------------- | --------- | ---------------------------------------- | ------------------ |
| 1_102_006_001 | CART_SKU_LIMIT_EXCEEDED      | 400       | 购物车商品种类已达上限50个               | 提示用户清理购物车 |
| 1_102_006_002 | CART_ITEM_NOT_EXISTS         | 400       | 购物车商品不存在                         | 刷新购物车         |
| 1_102_006_003 | CART_QUANTITY_INVALID        | 400       | 购买数量不符合要求（起订量/递增量/库存） | 按提示修正数量     |
| 1_102_006_004 | CART_AGREEMENT_NOT_EXISTS    | 400       | 挂牌不存在                               | 刷新页面           |
| 1_102_006_005 | CART_SKU_NOT_EXISTS          | 400       | 所选SKU不存在                            | 刷新页面           |
| 1_102_006_006 | CART_SKU_NO_STOCK            | 400       | 库存不足，请重新选择                     | 修改购买数量或移除 |
| 1_102_006_007 | CART_SKU_STOCK_LESS_THAN_MOQ | 400       | 库存数量小于起订量，无法下单             | 移除该SKU          |
| 1_102_006_008 | CART_CHECKOUT_NO_SELECTION   | 400       | 请选择挂牌产品                           | 勾选SKU后重试      |
| 1_102_006_009 | CART_CHECKOUT_NOT_TRADE_TIME | 400       | 未在交易时间段内，无法下单               | 等待交易时间       |

### 去结算 / 提交订单错误码

| 错误码                           | 名称           | HTTP 状态 | 描述                                        | 处理建议             |
| -------------------------------- | -------------- | --------- | ------------------------------------------- | -------------------- |
| CART_CHECKOUT_VALIDATION_FAILED  | 去结算校验失败 | 400       | 基于 1.4.51 价格/库存/状态校验不通过        | 按提示修正后重试     |
| CHECKOUT_TOKEN_EXPIRED           | token 过期     | 410       | 结算token已超过30分钟有效期                 | 返回购物车重新去结算 |
| CHECKOUT_TOKEN_INVALID           | token 无效     | 400       | token 不存在或不匹配当前用户                | 刷新后重新发起结算   |
| ORDER_SUBMIT_VALIDATION_FAILED   | 下单校验失败   | 400       | 二次校验不通过（仅校验库存/状态，不含价格） | 按提示修正后重试     |
| ORDER_SUBMIT_IDEMPOTENT_CONFLICT | 重复提交       | 409       | 幂等冲突/重复提交                           | 刷新订单状态         |
| INTERNAL_SERVER_ERROR            | 系统异常       | 500       | 内部错误                                    | 稍后重试或联系管理员 |

---

## 枚举定义

> 所有枚举值使用 Integer 数字编码，与后端枚举类 `code` 字段一致。

| 枚举名          | 基础类型 | 枚举值 | 含义     | 后端枚举类                                           |
| --------------- | -------- | ------ | -------- | ---------------------------------------------------- |
| ContractType    | int      | 1      | 纸质合同 | OrderEnums.OrderContractTypeEnum.PAPER_CONTRACT      |
| ContractType    | int      | 2      | 电子合同 | OrderEnums.OrderContractTypeEnum.ELECTRONIC_CONTRACT |
| ContractType    | int      | 3      | 不签合同 | OrderEnums.OrderContractTypeEnum.NO_CONTRACT         |
| ListingStatus   | int      | 0      | 待交易   | AgreementSkuEnums.ListingStatusEnum.WAITING          |
| ListingStatus   | int      | 1      | 交易中   | AgreementSkuEnums.ListingStatusEnum.TRADING          |
| ListingStatus   | int      | 2      | 已下架   | AgreementSkuEnums.ListingStatusEnum.OFF_SHELF        |
| ListingStatus   | int      | 3      | 已售罄   | AgreementSkuEnums.ListingStatusEnum.SOLD_OUT         |
| SkuEnableStatus | int      | 0      | 禁用     | AgreementSkuEnums.SkuEnableStatus.DISABLED           |
| SkuEnableStatus | int      | 1      | 启用     | AgreementSkuEnums.SkuEnableStatus.ENABLED            |

---

## 1. 获取购物车数量

**GET** `/api/frontend/cart/count`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取

### 请求参数

无

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "cartItemCount": 12
  }
}
```

| 字段               | 类型         | 说明                          |
| ------------------ | ------------ | ----------------------------- |
| data.cartItemCount | int32(min=0) | 购物车 SKU 总条数，含所有状态 |

### 业务规则

1. 统计当前用户购物车中所有 SKU 种类数，包含全部状态（交易中、待交易、已售罄、已下架）
2. 前端展示：0 不显示红点，1~99 显示数字，>99 显示 99⁺

---

## 2. 查询购物车列表

**GET** `/api/frontend/cart`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取

### 请求参数

| 参数     | 来源  | 必填 | 类型                | 说明              |
| -------- | ----- | ---- | ------------------- | ----------------- |
| pageNo   | query | 否   | int32(min=1)        | 页码，默认 1      |
| pageSize | query | 否   | int32(min=1,max=50) | 每页条数，默认 20 |

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "list": [
      {
        "supplierId": 77001,
        "supplierName": "供应商A",
        "spuList": [
          {
            "spuId": 66001,
            "productThumbnail": "https://cdn.example.com/spu/66001-thumb.jpg",
            "productName": "某商品A",
            "deliveryTime": 3,
            "skuList": [
              {
                "skuCode": "SKU-001",
                "skuStatus": 1,
                "listingStatus": 1,
                "skuAttrCombination": "A1/B2/C3",
                "taxIncludedPrice": 50.0000,
                "stockQty": 1200,
                "purchaseQty": 100,
                "minOrderQty": 10,
                "stepQty": 5
              }
            ]
          }
        ]
      }
    ],
    "pageNo": 1,
    "pageSize": 20,
    "total": 1
  }
}
```

| 字段                                               | 类型                 | 说明                                               |
| -------------------------------------------------- | -------------------- | -------------------------------------------------- |
| data.list                                          | array(object)        | 供应商列表                                         |
| data.list[].supplierId                             | int64(>0)            | 供应商ID                                           |
| data.list[].supplierName                           | string(max=200)      | 供应商名称                                         |
| data.list[].spuList                                | array(object)        | SPU 列表                                           |
| data.list[].spuList[].spuId                        | int64(>0)            | SPU ID（即 agreementId）                           |
| data.list[].spuList[].productThumbnail             | string(url,max=500)  | 产品图片缩略图                                     |
| data.list[].spuList[].productName                  | string(max=200)      | 产品名称                                           |
| data.list[].spuList[].deliveryTime                 | int32 / null         | 预计发货时间（天），挂牌时填写                     |
| data.list[].spuList[].skuList                      | array(object)        | SKU 列表                                           |
| data.list[].spuList[].skuList[].skuCode            | string(max=64)       | SKU 编码                                           |
| data.list[].spuList[].skuList[].skuStatus          | int(SkuEnableStatus) | SKU 启用状态（0=禁用，1=启用）                     |
| data.list[].spuList[].skuList[].listingStatus      | int(ListingStatus)   | 挂牌状态（0=待交易，1=交易中，2=已下架，3=已售罄） |
| data.list[].spuList[].skuList[].skuAttrCombination | string(max=500)      | SKU 属性值组合                                     |
| data.list[].spuList[].skuList[].taxIncludedPrice   | decimal(16,4)        | 挂牌价（含税），取实时价格                         |
| data.list[].spuList[].skuList[].stockQty           | int32(min=0)         | 库存数量，取实时库存                               |
| data.list[].spuList[].skuList[].purchaseQty        | int32(min=1)         | 购买数量                                           |
| data.list[].spuList[].skuList[].minOrderQty        | int32(min=0)         | 起订量                                             |
| data.list[].spuList[].skuList[].stepQty            | int32(min=0)         | 递增量                                             |
| data.pageNo                                        | int32(min=1)         | 页码                                               |
| data.pageSize                                      | int32(min=1)         | 每页条数                                           |
| data.total                                         | int64(min=0)         | 总条数                                             |

### 排序规则

- 供应商层：MAX(item.createTime) DESC — 该供应商下最近一次加购时间倒序
- SPU 层：MAX(item.createTime) DESC — 该 SPU 下最近一次加购时间倒序
- SKU 层：状态优先（TRADING > PENDING > SOLD_OUT > OFF_SHELF），再 createTime DESC

### 业务规则

1. 购物车明细从本地 DB 读取，实时数据（价格、库存、状态、起订量、递增量）从 AgreementApi 并行获取
2. SKU 处于非"交易中"时，前端仅展示属性值组合和挂牌价，其余字段不展示（后端仍返回全量数据，展示控制由前端处理）
3. 分页 total 为供应商维度的分组总数

---

## 3. 加入购物车

**POST** `/api/frontend/cart/items`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取

### 请求体

```json
{
  "skuCode": "SKU-001",
  "purchaseQty": 10
}
```

| 字段        | 类型           | 必填 | 说明     |
| ----------- | -------------- | ---- | -------- |
| skuCode     | string(max=64) | 是   | SKU 编码 |
| purchaseQty | int32(min=1)   | 是   | 加购数量 |

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "cartItemCount": 12,
    "added": true
  }
}
```

| 字段               | 类型         | 说明                                        |
| ------------------ | ------------ | ------------------------------------------- |
| data.cartItemCount | int32(min=0) | 购物车 SKU 总条数                           |
| data.added         | boolean      | 是否新增 SKU（false 表示已有 SKU 累加数量） |

### 业务规则

1. 通过 skuCode 反查 AgreementApi 获取 agreementId、supplierId 等信息
2. 校验挂牌和 SKU 存在性
3. 校验 SKU 库存 >= 起订量（无货不可加购）
4. 购物车 SKU 种类 <= 50，超出返回 CART_SKU_LIMIT_EXCEEDED
5. 同 skuCode 已存在 → 累加 purchaseQty，`added = false`
6. 新 SKU → 创建购物车明细记录，`added = true`
7. 返回购物车当前 SKU 种类总数

---

## 4. 调整购物车 SKU 数量

**PATCH** `/api/frontend/cart/items/{skuCode}/quantity`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取

### 请求参数

| 参数        | 来源 | 必填 | 类型           | 说明                             |
| ----------- | ---- | ---- | -------------- | -------------------------------- |
| skuCode     | path | 是   | string(max=64) | SKU 编码                         |
| purchaseQty | body | 是   | int32(min=1)   | 调整后购买数量（绝对值，非增量） |

### 请求体

```json
{
  "purchaseQty": 110
}
```

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "skuCode": "SKU-001",
    "purchaseQty": 110,
    "subTotalAmount": 5500.0000
  }
}
```

| 字段                | 类型           | 说明                                       |
| ------------------- | -------------- | ------------------------------------------ |
| data.skuCode        | string(max=64) | SKU 编码                                   |
| data.purchaseQty    | int32(min=1)   | 最新购买数量                               |
| data.subTotalAmount | decimal(16,4)  | 小计金额（purchaseQty × taxIncludedPrice） |

### 业务规则

1. 通过 skuCode + 登录上下文定位购物车明细记录
2. 校验购物车明细存在且属于当前用户
3. purchaseQty 必须为正整数
4. purchaseQty >= minOrderQty
5. (purchaseQty - minOrderQty) % stepQty == 0
6. purchaseQty <= stockQty
7. 校验不通过返回 CART_QUANTITY_INVALID（message 含具体原因）

---

## 5. 批量删除 SKU

**POST** `/api/frontend/cart/items/batch-delete`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取
- 使用约定：单个 SKU 删除也走本接口（`skuCodes` 仅传 1 个元素）

### 请求体

```json
{
  "skuCodes": ["SKU-001", "SKU-002", "SKU-003"]
}
```

| 字段     | 类型                   | 必填 | 说明                |
| -------- | ---------------------- | ---- | ------------------- |
| skuCodes | array(string, max=100) | 是   | 待删除 SKU 编码列表 |

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "deletedCount": 3,
    "failedSkuCodes": [],
    "cartItemCount": 7
  }
}
```

| 字段                | 类型          | 说明                                          |
| ------------------- | ------------- | --------------------------------------------- |
| data.deletedCount   | int32(min=0)  | 成功删除数量                                  |
| data.failedSkuCodes | array(string) | 删除失败的 SKU 编码（不存在或不属于当前用户） |
| data.cartItemCount  | int32(min=0)  | 删除后购物车 SKU 总条数                       |

### 业务规则

1. 仅删除属于当前用户的购物车明细
2. 不存在或不属于当前用户的 skuCode 加入 failedSkuCodes，不中断整体删除
3. 软删除（逻辑删除）

---

## 6. 去结算

**POST** `/api/frontend/cart/checkout`

- 认证方式：登录态（Bearer Token）
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取
- 校验逻辑：调用云商 `1.4.51` 批量查询挂牌 SKU 价格/库存/状态接口，执行去结算校验（价格+库存+状态）

### 请求体

```json
{
  "addressId": 90001,
  "items": [
    {
      "skuCode": "SKU-001",
      "purchaseQty": 10,
      "amount": 500.0000
    },
    {
      "skuCode": "SKU-002",
      "purchaseQty": 20,
      "amount": 1000.0000
    }
  ]
}
```

| 字段                | 类型                | 必填 | 说明                                                |
| ------------------- | ------------------- | ---- | --------------------------------------------------- |
| addressId           | int64(>0)           | 否   | 收货地址 ID                                         |
| items               | array(object)       | 是   | 参与结算的 SKU 列表                                 |
| items[].skuCode     | string(max=64)      | 是   | SKU 编码                                            |
| items[].purchaseQty | int32(min=1)        | 是   | 购买数量                                            |
| items[].amount      | decimal(16,4,min=0) | 是   | 当前 SKU 总金额（前端计算的小计，用于价格变动校验） |

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "token": "6ce47fa2-22b0-4e7f-8d2f-2cc5fef70f9f",
    "expireAt": "2026-04-13 15:45:00",
    "checkoutSummary": {
      "supplierCount": 2,
      "spuCount": 3,
      "skuCount": 5,
      "totalPurchaseQty": 230,
      "totalAmount": 12345.6700
    },
    "items": [
      {
        "supplierId": 77001,
        "supplierName": "供应商A",
        "supplierAmount": 10500.0000,
        "spuList": [
          {
            "spuId": 66001,
            "productName": "某商品A",
            "productThumbnail": "https://cdn.example.com/spu/66001-thumb.jpg",
            "deliveryTime": 3,
            "skuList": [
              {
                "skuCode": "SKU-001",
                "skuAttrCombination": "A1/B2/C3",
                "taxIncludedPrice": 50.0000,
                "purchaseQty": 100,
                "amount": 5000.0000,
                "stockQty": 1200,
                "minOrderQty": 10,
                "stepQty": 5
              }
            ]
          }
        ]
      }
    ]
  }
}
```

| 字段                                                | 类型                          | 说明                                                         |
| --------------------------------------------------- | ----------------------------- | ------------------------------------------------------------ |
| data.token                                          | string(uuid)                  | 结算 token，30 分钟有效                                      |
| data.expireAt                                       | datetime(yyyy-MM-dd HH:mm:ss) | token 失效时间                                               |
| data.checkoutSummary                                | object                        | 结算汇总信息                                                 |
| data.checkoutSummary.supplierCount                  | int32(min=0)                  | 供应商数量                                                   |
| data.checkoutSummary.spuCount                       | int32(min=0)                  | SPU 数量                                                     |
| data.checkoutSummary.skuCount                       | int32(min=0)                  | SKU 数量                                                     |
| data.checkoutSummary.totalPurchaseQty               | int32(min=0)                  | 合计购买数量（全部 SKU 的 purchaseQty 之和）                 |
| data.checkoutSummary.totalAmount                    | decimal(16,4)                 | 合计金额                                                     |
| data.items                                          | array(object)                 | 商品信息列表（按 供应商 → SPU → SKU 三层分组，供确认订单页渲染） |
| data.items[].supplierId                             | int64(>0)                     | 供应商 ID                                                    |
| data.items[].supplierName                           | string(max=200)               | 供应商名称                                                   |
| data.items[].supplierAmount                         | decimal(16,4)                 | 供应商小计金额（该供应商下全部 SKU amount 之和）             |
| data.items[].spuList                                | array(object)                 | SPU 列表                                                     |
| data.items[].spuList[].spuId                        | int64(>0)                     | SPU ID（即 agreementId）                                     |
| data.items[].spuList[].productName                  | string(max=200)               | 产品名称                                                     |
| data.items[].spuList[].productThumbnail             | string(url,max=500)           | 产品图片缩略图                                               |
| data.items[].spuList[].deliveryTime                 | int32 / null                  | 预计发货时间（天）                                           |
| data.items[].spuList[].skuList                      | array(object)                 | SKU 列表                                                     |
| data.items[].spuList[].skuList[].skuCode            | string(max=64)                | SKU 编码                                                     |
| data.items[].spuList[].skuList[].skuAttrCombination | string(max=500)               | SKU 属性值组合                                               |
| data.items[].spuList[].skuList[].taxIncludedPrice   | decimal(16,4)                 | 挂牌价（含税，实时价格）                                     |
| data.items[].spuList[].skuList[].purchaseQty        | int32(min=1)                  | 购买数量                                                     |
| data.items[].spuList[].skuList[].amount             | decimal(16,4)                 | 小计金额（挂牌价 × 购买数量）                                |
| data.items[].spuList[].skuList[].stockQty           | int32(min=0)                  | 库存数量                                                     |
| data.items[].spuList[].skuList[].minOrderQty        | int32(min=0)                  | 起订量                                                       |
| data.items[].spuList[].skuList[].stepQty            | int32(min=0)                  | 递增量                                                       |

### 业务规则

1. 校验至少选中 1 个 SKU，否则返回 CART_CHECKOUT_NO_SELECTION
2. 校验当前时间是否在平台交易时间段内（ConfigTradeOrderRuleApi），否则返回 CART_CHECKOUT_NOT_TRADE_TIME
3. 调用 AgreementApi（云商 1.4.51）并行获取实时 SKU 状态、库存、价格、起订量、递增量
4. 逐个 SKU 校验：
   - SKU 状态非"交易中" → CART_CHECKOUT_VALIDATION_FAILED（msg: 商品信息已变更）
   - 库存不足 → CART_CHECKOUT_VALIDATION_FAILED（msg: 库存不足）
   - 购买数量 < 起订量 → CART_CHECKOUT_VALIDATION_FAILED（msg: 购买数量必须大于等于起订量）
   - (购买数量 - 起订量) % 递增量 ≠ 0 → CART_CHECKOUT_VALIDATION_FAILED（msg: 购买数量不符合递增量规则）
   - 价格变动（前端传入 amount ≠ 实时价格 × 购买数量）→ CART_CHECKOUT_PRICE_CHANGED（msg: 商品价格发生变动，请返回购物车确认）
5. 全部校验通过后：
   - 生成结算 token（用户 ID + 来源 + 时间戳 + 随机串），全局唯一
   - 创建商品临时快照写入 Redis，关联 token（快照中的金额基于实时价格计算）
   - Token 有效期 30 分钟，一次性凭证
6. 返回 token、过期时间、汇总信息，以及按 供应商 → SPU → SKU 三层分组的 `items` 商品列表（供前端直接渲染确认订单页，无需再次调用购物车列表接口）

---

## 7. 提交订单

**POST** `/api/frontend/order/submit`

- 认证方式：登录态（Bearer Token）
- 幂等性：建议 Header 传 `Idempotency-Key`
- 上下文字段：`tenantId`、`companyId`、`userId` 由服务端从登录上下文提取
- 二次校验：调用云商 `1.4.51`，仅校验库存与状态（不校验价格）
- 拆单规则：按 `供应商 → 合同类型 → 合同模板ID → 结算节点` 拆分子母单
- 费用结算口径：拆单后先按 SKU 维度计算费用并截取到 8 位小数，再按子订单聚合并截取到 2 位小数（不四舍五入）
- 执行层约束：拆单规则由 `frontend-orch` 执行，`settlement-biz` 仅按拆分结果落库

### 请求体

```json
{
  "token": "6ce47fa2-22b0-4e7f-8d2f-2cc5fef70f9f",
  "addressId": 90001,
  "invoiceId": 50001,
  "contractForms": [
    {
      "agreementId": 88001,
      "contractType": 2,
      "contractTemplateId": 99001
    },
    {
      "agreementId": 88002,
      "contractType": 1,
      "contractTemplateId": 99002
    }
  ],
  "supplierRemarks": [
    {
      "supplierId": 77001,
      "remark": "请尽快发货"
    }
  ]
}
```

| 字段                               | 类型              | 必填 | 说明                               |
| ---------------------------------- | ----------------- | ---- | ---------------------------------- |
| token                              | string(uuid)      | 是   | 结算 token                         |
| addressId                          | int64(>0)         | 是   | 收货地址 ID                        |
| invoiceId                          | int64(>0)         | 否   | 发票 ID                            |
| contractForms                      | array(object)     | 是   | 合同形式列表（每个挂牌一项）       |
| contractForms[].agreementId        | int64(>0)         | 是   | 协议 ID（挂牌 ID）                 |
| contractForms[].contractType       | int(ContractType) | 是   | 合同类型（1=纸质，2=电子，3=不签） |
| contractForms[].contractTemplateId | int64(>0)         | 是   | 合同模板 ID                        |
| supplierRemarks                    | array(object)     | 否   | 供应商备注列表                     |
| supplierRemarks[].supplierId       | int64(>0)         | 是   | 供应商 ID                          |
| supplierRemarks[].remark           | string(max=100)   | 否   | 备注内容                           |

### 响应体

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "parentOrderNo": "PO4500022036008183389970433",
    "subOrderCount": 2,
    "subOrders": [
      {
        "subOrderNo": "OD4500022036008183389970434",
        "supplierId": 77001,
        "contractType": 2,
        "contractTemplateId": 99001,
        "settlementNode": "PAYMENT_AFTER_7D",
        "amount": 10000.00
      },
      {
        "subOrderNo": "OD4500022036008183389970435",
        "supplierId": 77002,
        "contractType": 1,
        "contractTemplateId": 99002,
        "settlementNode": "PAYMENT_AFTER_SIGN",
        "amount": 2345.67
      }
    ]
  }
}
```

| 字段                                | 类型              | 说明                                            |
| ----------------------------------- | ----------------- | ----------------------------------------------- |
| data.parentOrderNo                  | string(max=64)    | 母订单号                                        |
| data.subOrderCount                  | int32(min=1)      | 子订单数量                                      |
| data.subOrders                      | array(object)     | 子订单列表                                      |
| data.subOrders[].subOrderNo         | string(max=64)    | 子订单号                                        |
| data.subOrders[].supplierId         | int64(>0)         | 供应商 ID                                       |
| data.subOrders[].contractType       | int(ContractType) | 合同类型（1=纸质，2=电子，3=不签）              |
| data.subOrders[].contractTemplateId | int64(>0)         | 合同模板 ID                                     |
| data.subOrders[].settlementNode     | string(max=64)    | 结算节点                                        |
| data.subOrders[].amount             | decimal(16,2)     | 子订单金额（拆单后按 2 位小数截取，不四舍五入） |

### 业务规则

1. **第一步：核心必过校验**
   - 校验收货地址是否填写
   - 校验合同形式是否全部选择
2. **第二步：token 和快照校验**
   - 校验结算 token 有效性（存在、未过期、未使用、匹配当前用户）
   - 校验商品快照一致性（快照状态为"未使用"）
   - 校验不通过 → CHECKOUT_TOKEN_EXPIRED 或 CHECKOUT_TOKEN_INVALID
3. **第三步：二次校验（并行）**
   - 调用云商 1.4.51 获取实时 SKU 状态、库存（不校验价格）
   - SKU 非交易中 → ORDER_SUBMIT_VALIDATION_FAILED
   - 库存不足 → ORDER_SUBMIT_VALIDATION_FAILED
4. **第四步：拆单与落库**
   - 按 `供应商 → 合同类型 → 合同模板ID → 结算节点` 拆分
   - SKU 维度费用截取 8 位小数 → 子订单维度聚合截取 2 位小数
   - 将"商品临时快照"转为"订单正式快照"存储至订单表
   - 标记 token 已使用
   - 清除购物车中已下单的 SKU
5. 提交成功后返回母订单号和子订单列表

---

## 设计决策记录

| 决策                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| 前端不传 agreementId           | 加入购物车仅传 skuCode，后端通过 AgreementApi 反查 agreementId 和 supplierId；skuCode 在系统中全局唯一 |
| 单删合并到批量删除             | 前端统一使用批量删除接口，单个删除时 skuCodes 传 1 个元素    |
| 使用 skuCode 作为操作标识      | 修改数量、删除等操作均使用 skuCode（而非购物车明细 ID），简化前端调用 |
| 去结算 = 校验 + token 生成     | 合并为一个接口，校验通过即生成 token 和商品快照写入 Redis    |
| 列表结构调整                   | 响应从 agreementGroups 改为 spuList，对齐前端命名习惯；spuId 即 agreementId |
| 列表增加分页                   | 购物车上限 50 SKU，分页以供应商维度分组为单位                |
| 去结算校验逻辑由 orch 层承载   | orch 层负责调用 AgreementApi 并行校验，settlement-biz 仅提供数据读写 |
| 提交订单拆单逻辑由 orch 层承载 | frontend-orch 执行拆单规则，settlement-biz 仅按拆分结果落库  |