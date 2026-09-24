# 常见踩坑记录

## 1. syntax error, expect [, actual error, pos 0

**错误信息**：
```
syntax error, expect [, actual error, pos 0, fieldName null
```

**原因**：
某个应为数组的字段传了非数组值（字符串、对象、null 等）。

**常见场景**：

```js
// ❌ EmployeeField 传了字符串
{ "employeeField_xxx": "张三" }

// ❌ ImageField 传了对象
{ "imageField_xxx": {name: "photo.jpg", url: "https://..."} }

// ❌ AssociationFormField 传了字符串
{ "associationFormField_xxx": "FINST-ABC123" }
```

**修复**：
确保这些字段的值是数组：

```js
// ✅ EmployeeField
{ "employeeField_xxx": ["userId123"] }

// ✅ ImageField（URLSearchParams 方式）
{ "imageField_xxx": "[{\"name\":\"photo.jpg\",\"url\":\"https://...\"}]" }

// ✅ AssociationFormField
{ "associationFormField_xxx": [{"appType":"...","formUuid":"...","formType":"receipt","instanceId":"...","title":"..."}] }
```

---

## 2. syntax error, expect {, actual string

**错误信息**：
```
syntax error, expect {, actual string, pos 0, fieldName associationFormField_xxx
```

**原因**：
AssociationFormField 传了 formInstId 字符串，而非对象数组。

**错误示例**：
```js
{ "associationFormField_xxx": "FINST-ABC123" }
```

**修复**：
传完整对象数组：

```js
{
  "associationFormField_xxx": [{
    "appType": "APP_XXX",
    "formUuid": "FORM_YYY",
    "formType": "receipt",
    "instanceId": "FINST-ABC123",
    "title": "关联记录标题"
  }]
}
```

---

## 3. 日期组件值的格式错误，必须为时间戳

**错误信息**：
```
日期组件值的格式错误，必须为时间戳
```

**原因**：
DateField 传了日期字符串而非毫秒时间戳。

**错误示例**：
```js
{ "dateField_xxx": "2026-09-28" }
{ "dateField_xxx": "2026-09-28T00:00:00Z" }
```

**修复**：
传 13 位毫秒时间戳，先按业务时区转换：

```js
// 北京时间 2026-09-28 00:00:00
{ "dateField_xxx": new Date('2026-09-28T00:00:00+08:00').getTime() }
// 结果：1790524800000
```

---

## 4. 关联表单数据不完整

**错误信息**：
```
关联表单数据不完整，缺少 appType / formUuid / formType / instanceId / title
```

**原因**：
AssociationFormField 对象缺少必填字段。

**错误示例**：
```js
// ❌ 缺少 appType、formUuid、formType
{ "associationFormField_xxx": [{"instanceId": "FINST-ABC123"}] }

// ❌ 字段名写错（formInstId vs instanceId）
{ "associationFormField_xxx": [{"appType":"APP_XXX","formUuid":"FORM_YYY","formType":"receipt","formInstId":"FINST-ABC123","title":"标题"}] }
```

**修复**：
补全 5 个必填字段，注意 `instanceId` 不是 `formInstId`：

```js
{
  "associationFormField_xxx": [{
    "appType": "APP_XXX",
    "formUuid": "FORM_YYY",
    "formType": "receipt",
    "instanceId": "FINST-ABC123",  // ← 注意：是 instanceId
    "title": "标题"
  }]
}
```

---

## 5. ImageField 传了 URL 字符串

**错误信息**：
```
上传图片格式错误
```

**原因**：
ImageField 传了 URL 字符串或 URL 数组，而非对象数组。

**错误示例**：
```js
// ❌ 传 URL 字符串
{ "imageField_xxx": "https://img.cdn1.vip/i/xxx.webp" }

// ❌ 传 URL 数组
{ "imageField_xxx": ["https://img.cdn1.vip/i/xxx.webp"] }
```

**修复**：
传对象数组（CLI / JSON body 方式）或 JSON 字符串化的对象数组（URLSearchParams 方式）：

```js
// CLI / JSON body 方式
{ "imageField_xxx": [{"name": "photo.jpg", "url": "https://img.cdn1.vip/i/xxx.webp"}] }

// URLSearchParams 方式（oyd.jsx 自定义页面）
{ "imageField_xxx": "[{\"name\":\"photo.jpg\",\"url\":\"https://img.cdn1.vip/i/xxx.webp\"}]" }
```

---

## 6. EmployeeField 传了姓名数组

**错误信息**：
```
成员字段值格式错误
```

**原因**：
EmployeeField 传了姓名数组而非 userId 字符串数组。

**错误示例**：
```js
{ "employeeField_xxx": ["张三", "李四"] }
```

**修复**：
传 userId 字符串数组：

```js
{ "employeeField_xxx": ["0252163856461532927016", "0252163856461532927017"] }
```

**获取 userId**：
```bash
# 通过钉钉 CLI 搜索用户
openyida dws contact user search --keyword "张三"

# 从已有数据中读取
openyida data query form APP_XXX FORM_XXX --size 1
# 返回的 formData 中 employeeField_xxx 包含 {name, userId}
# 提取 userId 即可
```

---

## 7. 查询返回的 _id 后缀字段无法回写

**问题**：
从 `searchFormDatas` 返回的 `formData` 中，某些字段带 `_id` 后缀（如 `associationFormField_xxx_id`），直接复制回 `saveFormData` 报错。

**原因**：
`_id` 后缀字段是**查询投影出的只读派生字段**，不能用于写入。

**错误示例**：
```js
// 从查询结果中直接复制
var data = searchResult.formData;
payload.associationFormField_xxx_id = data.associationFormField_xxx_id;
```

**修复**：
使用不带 `_id` 后缀的原始字段名，并重新构造写入格式：

```js
// 查询返回
{
  "associationFormField_xxx_id": "\"[{...}]\""  // 只读派生字段
}

// 写入时需要重新构造
payload.associationFormField_xxx = [{
  appType: '...',
  formUuid: '...',
  formType: 'receipt',
  instanceId: '...',
  title: '...'
}];
```

---

## 8. CSRF Token 缺失导致 403

**错误信息**：
```
403 Forbidden
```

**原因**：
在 oyd.jsx 自定义页面中用 `fetch` 调用 API 时，未带 CSRF token。

**修复**：
从 cookie 或全局变量中获取 CSRF token，并在请求头中带上：

```js
function getCsrfToken() {
  // 1. document.cookie
  var m = document.cookie.match(/(?:^|;\s*)csrfToken=([^;]*)/);
  if (m) return m[1];
  m = document.cookie.match(/(?:^|;\s*)XSRF-TOKEN=([^;]*)/);
  if (m) return m[1];
  // 2. window.g_config
  if (window.g_config && window.g_config._csrf_token) return window.g_config._csrf_token;
  // 3. meta tag
  var meta = document.querySelector('meta[name="_csrf_token"],meta[name="csrf-token"]');
  if (meta) return meta.getAttribute('content') || '';
  // 4. window 全局
  if (window._csrf_token) return window._csrf_token;
  return '';
}

var csrf = getCsrfToken();

fetch('/dingtalk/web/' + appType + '/v1/form/saveFormData.json', {
  method: 'POST',
  headers: {
    'x-csrf-token': csrf,
    'x-xsrf-token': csrf
  },
  body: new URLSearchParams({...})
});
```

---

## 9. 图床上传后图片无法预览

**问题**：
上传图片到图床后，在 ImageField 中无法预览。

**原因**：
ImageField 对象缺少 `previewUrl` 或 `downloadUrl` 字段。

**修复**：
确保图片对象包含 `url`、`previewUrl`、`downloadUrl`：

```js
var photo = {
  name: 'photo.jpg',
  size: 12345,
  url: 'https://img.cdn1.vip/i/xxx.webp',
  previewUrl: 'https://img.cdn1.vip/i/xxx.webp',  // 预览 URL
  downloadUrl: 'https://img.cdn1.vip/i/xxx.webp'  // 下载 URL
};
```

---

## 10. openyida CLI 校验通过但 API 仍报错

**问题**：
`openyida data create form` 命令没有报错，但实际数据未写入。

**原因**：
CLI 只校验基本格式，服务端可能有额外限制（如字段权限、公式校验、业务规则）。

**排查方法**：

1. 加 `--json` 查看完整请求和响应：
```bash
openyida data create form APP_XXX FORM_XXX \
  --expect-form-name "表单名" --expect-form-type receipt \
  --data-json '{...}' --json
```

2. 检查响应中的 `errorMsg` 或 `errorLevel` 字段。

3. 在宜搭表单设计器中检查字段权限（是否只读、隐藏）。

4. 检查是否有公式字段或业务规则阻止写入。

---

## 11. 浏览器端调试时无法访问页面上下文（state / utils / $）

**问题**：
在 DevTools Console 中想调试 oyd.jsx 自定义页面的 `state`、`utils`、`$`（组件引用）等，但找不到入口。

**原因**：
oyd.jsx 页面的上下文被封装在 LeGao 框架中，不能直接通过全局变量访问。

**修复**：
通过 `window.LeGao.getContext().__debugThis__` 获取页面组件的 `this`：

```js
// 在 DevTools Console 中执行
var ctx = window.LeGao.getContext().__debugThis__;

// 访问页面 state
ctx.state;

// 访问页面 utils（自定义函数）
ctx.utils;

// 访问页面 $（组件引用）
ctx.$;

// 调用页面方法
ctx.someFunction();

// 查看/修改 state
console.log(ctx.state.checkinDialog);
ctx.setState({ /* ... */ });
```

**快捷写法**（Console 里一行搞定）：

```js
var _ = window.LeGao.getContext().__debugThis__;
// 之后直接用 _.state、_.utils、_.$ 访问
```

**注意事项**：
- `__debugThis__` 仅在开发/调试环境可用，生产环境可能被移除。
- 此方法获取的是自定义页面根组件的 `this`，包含所有在 oyd.jsx 中定义的 state、方法和生命周期。
- 如果页面有多个自定义页面组件嵌套，`getContext()` 返回的是当前活跃页面的上下文。

---

## 调试清单

遇到字段格式问题时，按以下顺序排查：

1. **确认字段类型**：在宜搭表单设计器中查看字段 ID 和类型。
2. **对照速查表**：检查 [field-formats.md](field-formats.md) 中的正确格式。
3. **本机 CLI 验证**：用 `openyida data create form` 测试最简 payload。
4. **打印实际 payload**：在代码中 `console.log(JSON.stringify(payload))`，复制到 CLI 验证。
5. **检查 CSRF token**：浏览器 fetch 调用时确保带上 token。
6. **查看完整响应**：加 `--json` 或 `console.log(response)` 查看错误详情。
