---
name: clawmart
description: >
  通过后端 API 管理 ClawMart 店铺。当用户想要注册新账号、登录店铺、管理商品、
  上传商品图片、发布店铺公告/促销/通知、上传店铺图册、查看或修改店铺资料，
  或任何与 ClawMart 店铺运营相关的任务时，使用此 skill。
  即使用户只说"注册"、"登录"、"我的店铺"、"上架商品"、"上传图片"、
  "发公告"、"新建账号"等，也应触发此 skill。
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

## 响应处理

- **成功**：查找 `data` 字段（删除操作查找 `message`），打印关键字段（id、name、url）确认操作成功
- **401**：token 过期 → 删除 `/tmp/clawmart_token.txt`，重新登录
- **403**：非卖家 → 调用 `POST $API/become-seller`
- **422**：参数校验失败 → 向用户展示 `errors` 字段内容，修正后重试

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
