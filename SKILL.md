---
name: clawmart
description: >
  通过后端 API 管理 ClawMart 店铺。当用户想要注册新账号、登录店铺、管理商品、
  上传商品图片、发布店铺公告/促销/通知、上传店铺图册、查看或修改店铺资料、
  查看店铺公开网址、开启或关闭店铺公开状态，
  或任何与 ClawMart 店铺运营相关的任务时，使用此 skill。
  即使用户只说"注册"、"登录"、"我的店铺"、"上架商品"、"上传图片"、
  "发公告"、"新建账号"、"我的店铺网址"、"店铺链接"等，也应触发此 skill。
  用户随口说出店铺动态也应触发，例如"今天猪肉涨价了"、"苹果卖完了"、
  "明天休息"、"今天全场9折"、"下周搞活动"等，直接记录为店铺公告。
  用户说经营记录也应触发，例如"记一下"、"记账"、"进货了"、"今天流水"、
  "花了多少钱"、"员工请假"、"设备坏了"等，直接记录为店铺日志。
---

# ClawMart 店铺管理

帮助用户通过后端 API 管理 ClawMart 店铺，涵盖：注册/登录、店铺资料、商品管理、
公告/通知、店铺图册上传、商品图片上传。

## API 地址

默认：`https://www.clawmart.cn/api`

如果用户使用自托管实例，询问其 API 地址。设置变量供后续复用：
```bash
API="https://www.clawmart.cn/api"  # 或用户自己的地址
```

---

## 第一步 — 注册 / 登录

所有受保护的接口都需要 `Authorization: Bearer <token>` 请求头。

如果用户想**注册新账号**，走注册流程；否则直接登录。

### 注册流程

逐步询问用户（每次只问一个）：

1. **用户名**：字母、数字、下划线、短横线或中文，最长 50 字符，服务器会验证唯一性
2. **密码**：至少 8 位
3. 确认密码（与密码相同，用于 `password_confirmation`）

然后注册：
```bash
curl -s -X POST "$API/register" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "username": "用户输入的用户名",
    "password": "用户输入的密码",
    "password_confirmation": "用户输入的密码"
  }'
```

返回示例：
```json
{"message":"User registered successfully","token":"2|xyz...","user":{"id":5,"username":"...","is_seller":false}}
```

保存 token，然后**自动调用成为卖家**接口（会同时创建店铺）：
```bash
TOKEN="返回的token值"
echo -n "$TOKEN" > /tmp/clawmart_token.txt

curl -s -X POST "$API/become-seller" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

接着**询问店铺名称**：
> "请问您的店铺叫什么名字？"

设置店铺名：
```bash
curl -s -X PUT "$API/my-store" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"shop_name": "用户输入的店铺名"}'
```

完成后提示：「✅ 注册完成！店铺「xxx」已创建，欢迎使用 ClawMart。」

**422 错误**：显示 `errors` 字段内容——通常是用户名已被占用或密码太短，让用户重新输入再试。

---

### 登录

登录使用 `username` 字段（不是 email）：
```bash
curl -s -X POST "$API/login" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"username":"用户名","password":"密码"}'
```

返回示例：
```json
{"message":"Login successful","token":"1|abc...","user":{"id":1,"is_seller":true}}
```

保存 token 供后续使用：
```bash
echo -n "token值" > /tmp/clawmart_token.txt
TOKEN=$(cat /tmp/clawmart_token.txt)
```

**用户必须有 `is_seller: true`**。若为 false，调用：
```bash
curl -s -X POST "$API/become-seller" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

**错误处理：**
- **401**：token 已过期 → 删除 `/tmp/clawmart_token.txt`，重新登录
- **403**：不是卖家 → 调用 `/become-seller`
- **422**：参数错误 → 显示 `errors` 字段，让用户修正后重试

---

## 第二步 — 店铺资料

获取店铺资料（卖家首次调用时自动创建）：
```bash
curl -s "$API/my-store" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

更新店铺资料：
```bash
curl -s -X PUT "$API/my-store" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "shop_name": "我的店铺",
    "description": "店铺简介",
    "contact_phone": "138xxxxxxxx",
    "address": "北京市朝阳区XX路1号",
    "business_hours": "09:00-21:00",
    "tags": ["生鲜", "有机"],
    "store_type": "retail",
    "is_public": true
  }'
```

`store_type` 可选值：`restaurant`（餐饮）| `retail`（零售）| `service`（服务）| `general`（综合）| `hotel`（住宿）

### 店铺公开网址

店铺公开页面地址格式为：
```
https://www.clawmart.cn/store/{slug}
```

`slug` 字段在 `GET $API/my-store` 的返回数据中可以找到。

**注意**：只有 `is_public: true` 时，公开页面才能被访客访问。若需开启，在更新店铺资料时传入 `"is_public": true`。

### 店铺二维码

二维码没有专门的 API，需通过以下步骤获取：

**第一步**：调用 `GET $API/my-store`，从返回数据中取出 `slug` 字段。

**第二步**：拼出二维码所指向的 URL（即店铺 AI 对话页）：
```
https://www.clawmart.cn/store/{slug}/chat
```

**第三步**：在终端生成二维码（需安装 `qrencode`）：
```bash
SLUG="从my-store接口获取的slug值"
QR_URL="https://www.clawmart.cn/store/${SLUG}/chat"

# 终端内直接显示二维码
qrencode -t UTF8 "$QR_URL"

# 或保存为图片文件
qrencode -o /tmp/store-qr.png "$QR_URL"
echo "二维码已保存到 /tmp/store-qr.png"
echo "二维码链接：$QR_URL"
```

若 `qrencode` 未安装：
```bash
apt-get install -y qrencode 2>/dev/null || brew install qrencode 2>/dev/null
```

完成后告知用户：「您的店铺二维码链接为：`{QR_URL}`，顾客扫码后可直接与您的 AI 店铺助手对话。」

---

## 第三步 — 商品管理

### 查看商品列表
```bash
curl -s "$API/my-store/products" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 新建商品
```bash
curl -s -X POST "$API/my-store/products" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "有机苹果",
    "price": 18.8,
    "stock": 500,
    "unit": "斤",
    "description": "山东烟台产，无农药，直接批发价",
    "is_active": true
  }'
```

### 修改商品
```bash
curl -s -X PUT "$API/my-store/products/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"price": 15.5, "stock": 300}'
```

### 删除商品
```bash
curl -s -X DELETE "$API/my-store/products/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

---

## 第四步 — 店铺公告 / 通知

### 随口录入（口语转公告）

用户可能随口说一句话，不会主动说"发公告"。只要内容涉及店铺动态，就应**自动判断 type 并调用 `POST $API/my-store/notes`**，无需用户确认字段。

| 用户说的话 | 判断 type | title 建议 |
|-----------|-----------|-----------|
| "今天猪肉涨价了" / "苹果卖完了" | `inventory` | 库存/价格更新 |
| "明天休息" / "下午3点开门" | `announcement` | 营业通知 |
| "今天全场9折" / "买二送一" | `discount` | 优惠活动 |
| "下周搞个亲子活动" | `event` | 活动预告 |
| 其他杂事 | `other` | 根据内容提炼 |

**处理流程**：
1. 从用户原话中提炼 `title`（10字以内）和 `content`（保留原意，可适当润色）
2. 判断 `type`
3. 直接调用接口，成功后回复：「✅ 已记录：{title}」

`type` 可选值：`discount`（折扣）| `announcement`（公告）| `inventory`（库存）| `event`（活动）| `other`（其他）

### 查看公告列表
```bash
curl -s "$API/my-store/notes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 发布公告
```bash
curl -s -X POST "$API/my-store/notes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "type": "discount",
    "title": "周末特惠",
    "content": "本周六日全场8折，欢迎光临！",
    "valid_from": "2026-04-19",
    "valid_until": "2026-04-20",
    "is_active": true
  }'
```

### 修改公告
```bash
curl -s -X PUT "$API/my-store/notes/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"is_active": false}'
```

### 删除公告
```bash
curl -s -X DELETE "$API/my-store/notes/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

---

## 第五步 — 图片上传

图片以 **multipart/form-data** 格式上传，单张最大 **5MB**。

上传前确认文件存在且大小合规：
```bash
ls -lh /path/to/image.jpg
```

### 店铺图册

`category` 可选值：`environment`（环境）| `menu`（菜单）| `products`（商品）| `video`（视频）

```bash
# 上传店铺图片
curl -s -X POST "$API/my-store/photos" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "image=@/path/to/photo.jpg" \
  -F "caption=店内环境" \
  -F "category=environment"

# 调整图片顺序（传入排序后的 id 数组）
curl -s -X PUT "$API/my-store/photos/reorder" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"ids": [3, 1, 2]}'

# 删除店铺图片
curl -s -X DELETE "$API/my-store/photos/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 商品图片

**第一张上传的图片**自动成为商品主图（显示在商品列表中），后续上传的图片进入图册。

```bash
# 上传商品图片
curl -s -X POST "$API/my-store/products/{productId}/photos" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "image=@/path/to/product.jpg" \
  -F "caption=正面图"

# 设置某张图片为主图
curl -s -X PUT "$API/my-store/product-photos/{photoId}/primary" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 删除商品图片（如删除主图，下一张自动升为主图）
curl -s -X DELETE "$API/my-store/product-photos/{photoId}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

---

## 第六步 — 店铺日志（私密经营记录）

日志是店主的**随手记**，五花八门，仅店主可见，**不会发布到公开动态**。什么都可以记：进货、流水、员工、设备、备忘、杂事……

`category` 可选值：`sales`（销售）| `purchase`（进货）| `operations`（运营）| `finance`（财务）| `staff`（人员）| `other`（其他，不确定时默认用这个）

`amount_type` 可选值：`income`（收入）| `expense`（支出）—— 有金额才填，没有不传

### 核心原则：直接记，别多问

店主随口说什么都直接记下来，**原文保留**，不改写、不润色。`category` 猜不准就用 `other`，有金额就顺手提取，没有就不填。**不要反复确认字段**。

记录成功后回复：「✅ 已记录」

### 日志 vs 公告的判断

| 特征 | 记为日志 | 记为公告 |
|------|---------|---------|
| 对象 | 自己看 | 给顾客看 |
| 典型内容 | 流水、进货、员工、设备、杂事 | 促销、折扣、营业通知、活动 |
| 例子 | "今天卖了3000" "冰柜坏了" "小李请假" | "今天全场8折" "明天休息" "下周搞活动" |

### 随口记录示例

```
用户："进了200斤苹果，花了560"
→ category=purchase, amount=560, amount_type=expense, content="进了200斤苹果，花了560"

用户："今天流水3200"
→ category=sales, amount=3200, amount_type=income, content="今天流水3200"

用户："冰柜坏了叫人修了，修理费200"
→ category=operations, amount=200, amount_type=expense, content="冰柜坏了叫人修了，修理费200"

用户："小李今天没来，让小张顶班"
→ category=staff, content="小李今天没来，让小张顶班"

用户："下午去银行存了钱"
→ category=other, content="下午去银行存了钱"
```

### API 调用
```bash
# 新增日志
curl -s -X POST "$API/my-store/logs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "content": "进了200斤苹果，花了560",
    "category": "purchase",
    "amount": 560,
    "amount_type": "expense"
  }'

# 查看日志列表
curl -s "$API/my-store/logs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 按分类筛选
curl -s "$API/my-store/logs?category=purchase" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 修改日志
curl -s -X PUT "$API/my-store/logs/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"content": "更正内容"}'

# 删除日志
curl -s -X DELETE "$API/my-store/logs/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

---

## 第七步 — 日志分析与经营建议

当用户问"最近经营怎么样"、"帮我分析一下"、"有什么建议"、"这个月赚了多少"等，执行以下流程：

**第一步**：拉取全部日志（多页时循环拉取直到无更多数据）：
```bash
curl -s "$API/my-store/logs?page=1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

**第二步**：读取日志数据后，结合以下维度做分析并输出建议：

| 分析维度 | 触发关键词 | 分析内容 |
|---------|----------|---------|
| 收支汇总 | "赚了多少" "流水" "收支" | 汇总 income / expense，算净利润 |
| 进货规律 | "进货" "备货" | 进货频率、品类、金额趋势，建议补货节奏 |
| 销售趋势 | "卖得怎样" "销售" | 对比不同时段流水，识别高峰/低谷 |
| 人员情况 | "员工" "人手" | 识别请假频率，提示人力风险 |
| 设备运营 | "设备" "维修" | 统计维修次数和费用，建议是否需要更换 |
| 综合分析 | "分析" "建议" "怎么样" | 覆盖所有维度，给出 3 条可操作建议 |

**输出格式**（综合分析时使用）：
```
📊 经营概览（近 X 条记录）
- 总收入：XXX 元  总支出：XXX 元  净利润：XXX 元
- 进货 X 次，销售记录 X 条，其他记录 X 条

📈 趋势观察
- [1–2 条规律性发现]

💡 建议
1. [具体可操作的建议]
2. [具体可操作的建议]
3. [具体可操作的建议]
```

**注意**：
- 数据量少（少于 5 条）时，诚实说明样本不足，仅做有限分析
- 有金额的日志才纳入收支统计，无金额的日志只做定性分析
- 不编造数据，只基于日志中实际记录的内容推断

---

## 第八步 — 进销存管理

每个店铺维护独立的进销存流水，通过累积流水计算当前库存。

`type` 可选值：`purchase`（进货/入库）| `sale`（销售/出库）| `adjustment`（盘点调整）| `waste`（损耗报废）

- **进货/调增** → `qty` 填**正数**
- **销售/损耗/调减** → `qty` 填**负数**

### 查看当前库存汇总
```bash
curl -s "$API/my-store/inventory/summary" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

返回每个品项的：`item_name`、`unit`、`stock`（当前结余数量）、`total_purchase`（累计进货金额）、`total_sale`（累计销售金额）

### 查看流水明细
```bash
# 全部流水
curl -s "$API/my-store/inventory/txns" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 按类型筛选
curl -s "$API/my-store/inventory/txns?type=purchase" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 按商品名搜索
curl -s "$API/my-store/inventory/txns?item=苹果" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 新增流水（随口录入）

用户说进货、出货、报废等，直接记录流水：

```
用户："进了100斤苹果，4块一斤"
→ type=purchase, item_name="苹果", qty=100, unit="斤", unit_price=4, total_amount=400

用户："苹果今天卖了60斤"
→ type=sale, item_name="苹果", qty=-60, unit="斤"

用户："胡萝卜烂了10斤扔掉了"
→ type=waste, item_name="胡萝卜", qty=-10, unit="斤"

用户："盘点一下，苹果实际还有25斤"（当前库存30斤）
→ type=adjustment, item_name="苹果", qty=-5, unit="斤", note="盘点调整"
```

```bash
curl -s -X POST "$API/my-store/inventory/txns" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "item_name": "苹果",
    "type": "purchase",
    "qty": 100,
    "unit": "斤",
    "unit_price": 4,
    "total_amount": 400,
    "note": "从批发市场进货"
  }'
```

### 修改 / 删除流水
```bash
# 修改
curl -s -X PUT "$API/my-store/inventory/txns/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"qty": 120, "total_amount": 480}'

# 删除
curl -s -X DELETE "$API/my-store/inventory/txns/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 库存查询与预警

用户问"苹果还有多少"、"帮我看看库存"时：
1. 调用 `GET /my-store/inventory/summary`
2. 按 `item_name` 过滤或展示全部
3. `stock ≤ 0` 时提示「⚠️ {item_name} 已售罄，建议补货」
4. `stock` 较低时（结合历史消耗速度判断）提示补货建议

---

- **成功**：查找 `data` 字段（删除操作查找 `message`），打印关键字段（id、name、url）确认操作成功
- **401**：token 过期 → 删除 `/tmp/clawmart_token.txt`，重新登录
- **403**：非卖家 → 调用 `POST $API/become-seller`
- **422**：参数校验失败 → 向用户展示 `errors` 字段内容，修正后重试

---

## 第九步 — CRM 客户管理

每个店铺可以管理自己的客户信息（买家/常客），记录联系方式、消费历史、互动记录。**仅店主可见，不对外公开。**

### 客户字段说明

| 字段 | 说明 |
|------|------|
| `name` | 客户姓名（必填）|
| `phone` | 手机号 |
| `email` | 邮箱 |
| `address` | 地址 |
| `note` | 备注 |
| `tags` | 标签数组，如 `["VIP","老客户"]` |
| `total_spent` | 累计消费金额（系统自动维护）|
| `visit_count` | 累计互动次数（系统自动维护）|
| `last_contact_at` | 最近联系时间（系统自动维护）|

### 互动类型 `type`

`visit`（到店）| `call`（电话）| `message`（微信/短信）| `order`（下单）| `other`（其他）

---

### API 调用

```bash
# 查看客户列表（支持 ?search= 模糊搜索姓名/手机）
curl -s "$API/my-store/customers" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 新增客户
curl -s -X POST "$API/my-store/customers" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "张三",
    "phone": "13800138000",
    "note": "常买苹果，每周来一次",
    "tags": ["老客户", "水果"]
  }'

# 修改客户
curl -s -X PUT "$API/my-store/customers/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"note": "已升级为VIP", "tags": ["VIP","老客户"]}'

# 删除客户
curl -s -X DELETE "$API/my-store/customers/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 新增互动记录（自动累计 visit_count / total_spent / last_contact_at）
curl -s -X POST "$API/my-store/customers/{customerId}/interactions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "type": "order",
    "content": "买了苹果5斤、胡萝卜3斤",
    "amount": 76.5
  }'

# 查看某客户的互动记录
curl -s "$API/my-store/customers/{customerId}/interactions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 删除互动记录
curl -s -X DELETE "$API/my-store/customer-interactions/{id}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"
```

### 随口录入示例

```
用户："帮我记一下，张三刚才买了76块5"
→ 先搜索客户 GET /my-store/customers?search=张三
→ 找到后 POST /my-store/customers/{id}/interactions，type=order, content="购买", amount=76.5

用户："加个新客户，李四，电话13912345678，是老客户"
→ POST /my-store/customers，name="李四", phone="13912345678", tags=["老客户"]

用户："帮我查一下张三买了多少次了"
→ GET /my-store/customers?search=张三，展示 visit_count 和 total_spent
```

### CRM 分析

用户问"哪些是老客户"、"谁买得最多"、"帮我整理一下客户"时：
1. 调用 `GET /my-store/customers` 拉取全部客户
2. 按 `total_spent` 降序 → 消费最多的客户
3. 按 `visit_count` 降序 → 最活跃的客户
4. `last_contact_at` 超过 30 天未联系 → 提示"流失风险客户"

---

## 海报生成

调用 AI 自动生成宣传海报图片，返回可直接使用的 CDN 图片 URL。**需登录**，免费/付费用户均可使用。

### 生成前主动询问参考图

触发海报生成后，**先问一句**再调用接口：

> 「好的！请问有没有想参考的图片？可以把图片文件直接发给我（支持 JPG/PNG），没有的话我直接根据描述生成。」

- 用户发了图片 → 读取文件，转 base64，放入 `objectImages`
- 用户说"没有" / 直接回复"生成吧" → 跳过，直接用 prompt 生成

**注意**：只在第一次触发时问一次，不要反复询问。

### 随口触发

用户说以下任何内容都应触发海报生成：

| 用户说的话 | 自动提取的 prompt |
|-----------|-----------------|
| "帮我做个海报，今天全场8折" | "生鲜门店促销海报，今天全场8折，色彩鲜明，现代简约风格" |
| "做个周末活动的宣传图" | "周末活动宣传海报，温暖活泼风格，适合社交媒体" |
| "帮我生成一张商品主图，苹果" | "新鲜苹果商品主图，白色背景，精美食品摄影风格" |
| "做个节日海报" | "节日促销海报，喜庆氛围，适合门店宣传" |

**如果用户的描述比较简单**，根据已知的店铺信息（店名、品类）自动补充 prompt，让海报更贴合场景。例如：用户说"做个8折海报"，已知是生鲜店，则自动拼成更完整的 prompt。

### API 调用

```bash
TOKEN=$(cat /tmp/clawmart_token.txt)

# 基础用法（纯文字生成海报）
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "prompt": "生鲜门店促销海报，今天全场8折，苹果、西瓜、蔬菜，鲜艳色彩，现代简约风",
    "aspectRatio": "1:1"
  }'

# 带参考图（用户发来的文件，multipart 上传）
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "prompt=参考这张图的风格，重新生成一张生鲜促销海报，加上今日特价大字" \
  -F "aspectRatio=1:1" \
  -F "object_files[]=@/path/to/reference.jpg"

# 带参考人物图（角色一致性）
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "prompt=店主形象宣传海报，温暖亲切风格" \
  -F "aspectRatio=9:16" \
  -F "character_files[]=@/path/to/person.jpg"

# 竖版（适合手机朋友圈/海报）
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "prompt": "门店开业宣传海报，竖版，大字标题，温暖橙色调",
    "aspectRatio": "9:16"
  }'

# 横版（适合店内屏幕/横幅）
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "prompt": "春季新品上市横幅广告，清新绿色调，16:9横版",
    "aspectRatio": "16:9"
  }'
```

**两种上传参考图方式：**

| 方式 | 字段名 | 格式 | 适用场景 |
|------|--------|------|---------|
| 文件上传（推荐） | `object_files[]` / `character_files[]` | multipart/form-data | 用户直接发图片文件 |
| base64 JSON | `objectImages` / `characterImages` | `[{"mimeType":"image/png","data":"base64..."}]` | 已有 base64 数据 |

**用户发来图片文件时的处理流程：**

```bash
# 用户发来图片保存到 /tmp/ref.jpg 后
curl -s -X POST "$API/my-store/poster" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "prompt=参考这张图生成促销海报，今日全场8折" \
  -F "aspectRatio=1:1" \
  -F "object_files[]=@/tmp/ref.jpg"
```

注意：使用文件上传时**不要**加 `Content-Type: application/json`，让 curl 自动设置 multipart 头。

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `prompt` | 必填 | 图片描述，越详细效果越好 |
| `aspectRatio` | `1:1` | 比例：`1:1`（正方形）、`9:16`（竖版）、`16:9`（横版）、`4:3`、`3:4` |
| `model` | `gemini-3.1-flash-image-preview` | 生成模型，一般保持默认 |
| `thinkingLevel` | `MINIMAL` | 思考深度，保持默认即可 |

### 成功响应

```json
{
  "success": true,
  "url": "https://cdn.clawmart.cn/posters/poster_abc123.png",
  "mimeType": "image/png",
  "prompt": "你的提示词",
  "model": "gemini-3.1-flash-image-preview"
}
```

生成完成后，将 `url` 返回给用户：

「✅ 海报生成完成！
🖼️ 图片链接：{url}
复制链接在浏览器打开即可查看/下载，也可直接发到微信群。」

### Prompt 撰写建议

当用户描述模糊时，按以下结构自动补全 prompt：

```
{内容主题}，{风格描述}，{色调偏好}，{用途场景}，{语言要求（如有中文文字需求）}
```

示例：
- 促销 → `"全场8折促销海报，色彩鲜艳，醒目大字，生鲜蔬果主题，现代简约风，中文文字"`
- 新品 → `"新品上市宣传图，清新自然，高端质感，白色背景，商品特写"`
- 节日 → `"春节/国庆节日促销，喜庆红色调，传统与现代结合，门店宣传海报"`

### 错误处理

- **502**：第三方生成服务超时或不可用 → 告知用户稍后重试
- **422**：参数校验失败（如 prompt 超 2000 字）→ 缩短 prompt 后重试
- 生成通常需要 **15–60 秒**，请提示用户耐心等待

---

## 预约管理（可选）

```bash
# 查看预约列表（可按状态筛选：pending/confirmed/in_progress/completed/cancelled）
curl -s "$API/my-store/bookings?status=pending" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# 更新预约状态
curl -s -X PUT "$API/my-store/bookings/{id}/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"status": "confirmed"}'
```
