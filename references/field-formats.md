# 宜搭 API 字段格式详细参考

## 基础字段

### TextField / TextareaField

```json
{ "textField_xxx": "文本内容" }
```

直接传字符串。

### NumberField

```json
{ "numberField_xxx": 42 }
```

传数字，不是字符串。

### DateField

```json
{ "dateField_xxx": 1790524800000 }
```

传 13 位毫秒时间戳。先按业务时区转换：

```js
new Date('2026-09-28T00:00:00+08:00').getTime() // 1790524800000
```

**不要**传 `"2026-09-28"` 或 `"2026-09-28T00:00:00Z"`。

### CascadeDateField（日期区间）

```json
{ "cascadeDateField_xxx": [1790524800000, 1790611200000] }
```

传毫秒时间戳数组，第一个是开始，第二个是结束。

### SelectField（单选）

```json
{ "selectField_xxx": "选项一" }
```

传选项文本字符串，不是数组。

### MultiSelectField / CheckboxField（多选）

```json
{ "checkboxField_xxx": ["选项一", "选项二"] }
```

传字符串数组。

---

## 复合字段

### EmployeeField（成员/员工）

```json
{ "employeeField_xxx": ["userId1", "userId2"] }
```

**传 userId 字符串数组**，不是姓名、不是对象。

#### 获取 userId

```bash
# 通过钉钉 CLI 搜索用户
openyida dws contact user search --keyword "张三"

# 从已有数据中读取
openyida data query form APP_XXX FORM_XXX --size 1
# 返回的 formData 中 employeeField_xxx 包含 {name, userId}
# 但写入时只传 userId 字符串数组
```

#### 常见错误

```json
// ❌ 错误：传姓名
{ "employeeField_xxx": ["张三"] }

// ❌ 错误：传对象
{ "employeeField_xxx": [{"name": "张三", "userId": "xxx"}] }

// ✅ 正确：传 userId 字符串数组
{ "employeeField_xxx": ["0252163856461532927016"] }
```

#### 查询返回格式

查询返回的 `formData` 中，EmployeeField 的值是对象数组：

```json
{
  "employeeField_xxx": [{"name": "张三", "userId": "0252163856461532927016"}]
}
```

写入时只提取 userId：

```js
var empIds = records.map(function(r) { return r.userId; });
payload.employeeField_xxx = empIds;
```

---

### DepartmentField（部门）

```json
{ "departmentField_xxx": ["deptId1", "deptId2"] }
```

传部门 ID 字符串数组。

---

### ImageField（图片）

ImageField 的格式**取决于调用方式**：

#### 方式一：浏览器 fetch + URLSearchParams（oyd.jsx 自定义页面）

ImageField 的值需要**再 JSON.stringify 一次**：

```js
var photos = [{
  name: 'photo.jpg',
  size: 12345,
  url: 'https://img.cdn1.vip/i/xxx.webp',
  previewUrl: 'https://img.cdn1.vip/i/xxx.webp',
  downloadUrl: 'https://img.cdn1.vip/i/xxx.webp'
}];

var formData = {
  imageField_xxx: JSON.stringify(photos)  // ← 注意：内层再序列化
};

fetch('/dingtalk/web/' + appType + '/v1/form/saveFormData.json', {
  method: 'POST',
  body: new URLSearchParams({
    formUuid: formUuid,
    formDataJson: JSON.stringify(formData)  // ← 外层序列化
  })
});
```

#### 方式二：openyida CLI

CLI 自动处理序列化，直接传对象数组：

```bash
openyida data create form APP_XXX FORM_XXX \
  --expect-form-name 打卡 --expect-form-type receipt \
  --data-json '{"imageField_xxx":[{"name":"photo.jpg","size":12345,"url":"https://img.cdn1.vip/i/xxx.webp"}]}'
```

#### 方式三：浏览器 fetch + application/json body

直接传对象数组，不需要二次序列化：

```js
fetch('/dingtalk/web/' + appType + '/v1/form/saveFormData.json', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    formUuid: formUuid,
    formDataJson: JSON.stringify({
      imageField_xxx: [{name: 'photo.jpg', size: 12345, url: 'https://...'}]
      // 直接写对象数组，不需要再 JSON.stringify
    })
  })
});
```

#### ImageField 对象字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 文件名 |
| `size` | 否 | 文件大小（字节） |
| `url` | 是 | 图片 URL |
| `previewUrl` | 否 | 预览 URL（可同 url） |
| `downloadUrl` | 否 | 下载 URL（可同 url） |

#### 常见错误

```json
// ❌ 错误：传 URL 字符串
{ "imageField_xxx": "https://img.cdn1.vip/i/xxx.webp" }

// ❌ 错误：传 URL 数组
{ "imageField_xxx": ["https://img.cdn1.vip/i/xxx.webp"] }

// ✅ 正确（URLSearchParams 方式）：JSON 字符串化的对象数组
{ "imageField_xxx": "[{\"name\":\"photo.jpg\",\"url\":\"https://...\"}]" }

// ✅ 正确（CLI / JSON body 方式）：对象数组
{ "imageField_xxx": [{"name": "photo.jpg", "url": "https://..."}] }
```

---

### AssociationFormField（关联表单）

```json
{
  "associationFormField_xxx": [{
    "appType": "APP_WRJWKV4Q4LACDC0DYIJX",
    "formUuid": "FORM-2D48383B65954853830E1A5ADE38BB6DEIMR",
    "formType": "receipt",
    "instanceId": "FINST-ABC123",
    "title": "2026/9/28"
  }]
}
```

**必须是对象数组**，每个对象包含 5 个必填字段：

| 字段 | 说明 |
|------|------|
| `appType` | 被关联表单所属应用的 appType |
| `formUuid` | 被关联表单的 formUuid |
| `formType` | 被关联表单类型，普通表单用 `"receipt"` |
| `instanceId` | 被关联数据的 formInstId（注意：字段名是 `instanceId`，不是 `formInstId`） |
| `title` | 被关联记录在关联组件中展示的标题 |

#### 获取关联信息

```bash
# 1. 查询被关联表单获取 formInstId
openyida data query form APP_XXX FORM_排班表 --size 1
# 返回: formInstId: "FINST-ABC123"

# 2. 从返回的 formData 中提取 title 信息
# formData.dateField_xxx 等字段可作为 title
```

#### 常见错误

```json
// ❌ 错误：传 formInstId 字符串
{ "associationFormField_xxx": "FINST-ABC123" }

// ❌ 错误：缺少必填字段
{ "associationFormField_xxx": [{"instanceId": "FINST-ABC123"}] }

// ❌ 错误：字段名写错（formInstId vs instanceId）
{ "associationFormField_xxx": [{"appType":"APP_XXX","formUuid":"FORM_XXX","formType":"receipt","formInstId":"FINST-ABC123","title":"标题"}] }

// ✅ 正确：完整对象数组
{ "associationFormField_xxx": [{"appType":"APP_XXX","formUuid":"FORM_XXX","formType":"receipt","instanceId":"FINST-ABC123","title":"标题"}] }
```

#### 查询返回格式

查询返回的 `formData` 中，关联表单字段以 `_id` 后缀显示：

```json
{
  "associationFormField_xxx_id": "\"[{...}]\""
}
```

`associationFormField_xxx_id` 是**只读派生字段**，不能复制回写入 payload。写入时必须使用不带 `_id` 后缀的原始字段 `associationFormField_xxx`。

---

### TableField（子表单/明细）

```json
{
  "tableField_xxx": [
    {"textField_a": "值1", "numberField_b": 10},
    {"textField_a": "值2", "numberField_b": 20}
  ]
}
```

传对象数组，每个对象是一行数据。

---

## 完整示例

### 执勤打卡提交（oyd.jsx 自定义页面）

```js
// 构建 payload
var p = {};

// 关联表单：指向排班记录
p['associationFormField_7lt687bcr'] = [{
  appType: 'APP_WRJWKV4Q4LACDC0DYIJX',
  formUuid: 'FORM-2D48383B65954853830E1A5ADE38BB6DEIMR',
  formType: 'receipt',
  instanceId: 'FINST-排班记录ID',
  title: '2026/9/28'
}];

// 日期
p['dateField_7lt6apgyz'] = 1790524800000;

// 文本
p['textField_7lt6b89ni'] = '操场';
p['textField_7lt6cc4d0'] = '眼保健操';

// 多行文本
p['textareaField_7lt6d9t8m'] = '情况说明...';

// 成员：userId 字符串数组
p['employeeField_7lt69t9rr'] = ['0252163856461532927016'];

// 图片：ImageField 在 URLSearchParams 方式下需要内层 JSON.stringify
p['imageField_7lt6ewnmy'] = JSON.stringify([
  {name: 'photo1.jpg', size: 12345, url: 'https://img.cdn1.vip/i/xxx.webp'}
]);

// 提交
fetch('/dingtalk/web/' + APP + '/v1/form/saveFormData.json', {
  method: 'POST',
  body: new URLSearchParams({
    formUuid: CF,
    formDataJson: JSON.stringify(p)
  })
}).then(function(r) { return r.json(); })
  .then(function(j) { console.log('提交结果:', j); });
```
