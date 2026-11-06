---
name: openyida-plus
description: "宜搭（Yida / 宜搭）开发必备 skill。只要任务涉及宜搭——包括但不限于自定义页面（oyd.jsx）、表单 API（saveFormData / updateFormData / searchFormDatas）、字段数据格式、发布、CLI 命令、CSS 适配、组件开发——就必须载入本 skill，不得跳过。"
user-invocable: true
argument-hint: [字段名或错误信息]
---

# openyida-plus

通过 openyida CLI 或浏览器 `fetch` 调用宜搭数据 API（`saveFormData`、`updateFormData`、`searchFormDatas` 等）时，各字段类型的正确数据格式、常见陷阱与本机调试方法。

## ⚠️ 强制规则

### 1. 载入条件：凡涉及宜搭，必须载入

只要任务涉及宜搭平台——包括但不限于：
- 自定义页面开发（oyd.jsx / yida.jsx）
- 表单数据 API（saveFormData / updateFormData / searchFormDatas / getFormDataById）
- 字段数据格式（ImageField / EmployeeField / AssociationFormField / DateField 等）
- openyida CLI 命令（publish / data / create-form / list-forms 等）
- 宜搭 CSS 适配、组件开发、数据源绑定
- 任何 aliwork.com 域名下的开发工作

**必须载入本 skill**，不得跳过。

### 2. 完成标准：必须发布，禁止糊弄

**宜搭相关代码改动完成后，必须执行 `openyida publish` 发布到宜搭平台。**

禁止以下行为：
- ❌ 仅执行 `npm run build` 就声称完成
- ❌ 仅执行 `git commit` 就声称完成
- ❌ 仅修改 src/build/dist 文件就声称完成
- ❌ 告诉用户"代码已改好，请手动发布"

**正确流程**：
1. 修改 `src/schedule-main.oyd.jsx`（源文件）
2. 同步到 `build/schedule-main.yida.jsx`（构建文件）
3. 同步到 `dist/schedule-main.yida.js`（压缩文件）
4. **执行 `openyida publish` 发布到宜搭平台**
5. 返回发布结果（version 号、访问 URL）

**发布命令示例**：
```bash
npx openyida publish pages/build/schedule-main.yida.jsx APP_WRJWKV4Q4LACDC0DYIJX FORM-6268233FD7694351ACDD6FA2E2131D802B24 --json
```

只有当 `openyida publish` 返回 `"success": true` 时，任务才算完成。

### 3. 自定义页面任务路由

任务涉及 oyd.jsx 自定义页面的架构、shadcn/ui 设计语言、CSS/Tailwind 适配、组件实现或页面布局时，**必须读取 [references/custom-page/README.md](references/custom-page/README.md)**，并遵循该专题的约束与发布工作流；表单字段的数据格式仍以本 skill 的 `references/field-formats.md` 为准，发布仍须满足上文「完成标准」。

## 使用时机

- 调用 `saveFormData` / `updateFormData` / `searchFormDatas` 报错 `syntax error, expect X, actual Y`
- 不确定某个字段（ImageField / EmployeeField / AssociationFormField / DateField ……）应该传什么格式
- 查询返回的字段值无法直接回写到写入接口
- 需要通过 `openyida data create form` 在本机验证字段格式

## 核心原则

### 1. formDataJson 是字符串

`saveFormData` 的 `formDataJson` 参数是 **JSON 字符串**，不是对象。

```bash
# CLI
openyida data create form APP_XXX FORM_XXX \
  --expect-form-name 打卡 --expect-form-type receipt \
  --data-json '{"textField_xxx":"文本","dateField_xxx":1790524800000}'
```

```js
// 浏览器 fetch
fetch('/dingtalk/web/' + appType + '/v1/form/saveFormData.json', {
  method: 'POST',
  body: new URLSearchParams({
    formUuid: formUuid,
    formDataJson: JSON.stringify({
      textField_xxx: '文本',
      dateField_xxx: 1790524800000
    })
  })
});
```

### 2. 外层 JSON 值 vs 内层 JSON 字符串

宜搭 API 的 `formDataJson` 是一个 JSON 对象字符串。**普通字段**的值直接写 JSON 类型（字符串、数字、数组、对象）；但**部分复合字段**的值需要**再嵌套一层 JSON 字符串**——这取决于 API 端点的解析方式。

**关键区分**：

| 调用方式 | formDataJson | 复合字段值 |
|----------|-------------|-----------|
| `openyida data create form` CLI | CLI 自动 `JSON.stringify` | 直接写 JS 对象/数组，CLI 负责序列化 |
| 浏览器 `fetch` + `URLSearchParams` | 手动 `JSON.stringify` 一次 | ImageField 等需要**再 `JSON.stringify` 一次** |
| 浏览器 `fetch` + `application/json` body | 手动 `JSON.stringify` 一次 | 复合字段值也直接写对象/数组，**不需要**二次序列化 |

> **经验**：在 oyd.jsx 自定义页面中用 `fetch` + `URLSearchParams` 提交时，ImageField 的值必须 `JSON.stringify([...])` 成字符串后再放入外层对象。EmployeeField 和 AssociationFormField 则直接放数组对象即可。

### 3. 查询返回的 `_id` 后缀字段是只读派生字段

查询 `searchFormDatas` 返回的 `formData` 中，部分字段会带 `_id` 后缀（如 `associationFormField_xxx_id`、`employeeField_xxx_id`）。这些是**查询投影出的只读派生字段**，**绝不能**复制回 `saveFormData` / `updateFormData` 的 payload。写入时必须使用**不带** `_id` 后缀的原始字段名。

## 字段格式速查表

详细格式见 [references/field-formats.md](references/field-formats.md)。

| 组件类型 | 写入格式 | 常见错误 |
|---------|---------|---------|
| TextField / TextareaField | `"文本"` | 传数字或数组 |
| NumberField | `10` | 传字符串 `"10"` |
| DateField | `1790524800000`（13位毫秒时间戳） | 传日期字符串 `"2026-09-24"` |
| CascadeDateField | `[1790524800000, 1790611200000]` | 传字符串数组 |
| SelectField（单选） | `"选项一"` | 传数组 `["选项一"]` |
| MultiSelectField（多选） | `["选项一","选项二"]` | 传字符串 |
| EmployeeField | `["userId1","userId2"]`（userId 字符串数组） | 传姓名数组或对象数组 |
| DepartmentField | `["deptId1"]` | 传部门名称 |
| ImageField（fetch+URLSearchParams） | `JSON.stringify([{name,size,url,previewUrl}])`（**内层再序列化**） | 直接传对象数组（不序列化） |
| ImageField（CLI / JSON body） | `[{name,size,url,previewUrl}]`（直接对象数组） | 传字符串 URL |
| AssociationFormField | `[{appType,formUuid,formType,instanceId,title}]` | 传 formInstId 字符串 |
| TableField（子表单） | `[{textField_xxx:"值",...}]` | 传字符串 |

## 浏览器端调试：获取页面上下文（this / state / utils / $）

在 oyd.jsx 自定义页面中，经常需要在 DevTools Console 里访问页面的 `state`、`utils`、`$`（组件引用）等上下文来调试。

**方法**：通过 `window.LeGao.getContext().__debugThis__` 获取页面组件的 `this`：

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
```

之后直接用 `_.state`、`_.utils`、`_.$` 访问。

**注意事项**：
- `__debugThis__` 仅在开发/调试环境可用，生产环境可能被移除。
- 此方法获取的是自定义页面根组件的 `this`，包含所有在 oyd.jsx 中定义的 state、方法和生命周期。
- 如果页面有多个自定义页面组件嵌套，`getContext()` 返回的是当前活跃页面的上下文。

## 本机调试方法

当字段格式不确定时，用 `openyida data create form` 在本机直接调 API 验证：

```bash
# 1. 确认已登录
openyida auth status

# 2. 用最简 payload 试一条数据
openyida data create form APP_XXX FORM_XXX \
  --expect-form-name "表单名" --expect-form-type receipt \
  --data-json '{"textField_xxx":"测试"}'

# 3. 逐步加入复杂字段，观察报错信息
openyida data create form APP_XXX FORM_XXX \
  --expect-form-name "表单名" --expect-form-type receipt \
  --data-json '{"textField_xxx":"测试","employeeField_xxx":["userId123"],"associationFormField_xxx":[{"appType":"APP_XXX","formUuid":"FORM_YYY","formType":"receipt","instanceId":"FINST_ZZZ","title":"标题"}]}'
```

CLI 会在发请求前做格式校验，提前拦截明显错误（如 AssociationFormField 缺少必填字段）。如果 CLI 校验通过但 API 仍报错，说明是服务端格式问题——此时用 `--json` 查看完整请求和响应。

## 常见错误与修复

详细踩坑记录见 [references/common-pitfalls.md](references/common-pitfalls.md)。

| 错误信息 | 原因 | 修复 |
|---------|------|------|
| `syntax error, expect [, actual error, pos 0, fieldName null` | 某个应为数组的字段传了非数组值 | 检查 EmployeeField / ImageField / AssociationFormField 是否为数组 |
| `syntax error, expect {, actual string` | 某个应为对象的字段传了字符串 | 检查 AssociationFormField 是否传了 formInstId 字符串而非对象数组 |
| `字段值的格式错误` | 字段值类型不对 | 对照速查表检查格式 |
| `日期组件值的格式错误，必须为时间戳` | DateField 传了日期字符串 | 改为 13 位毫秒时间戳 |
| `关联表单数据不完整` | AssociationFormField 缺少必填字段 | 补全 appType / formUuid / formType / instanceId / title |

## 工具约束

- 本机调试使用 `openyida data create form` 命令，需先 `openyida login` 或确认 `openyida auth status` 正常。
- 浏览器端调试可在 DevTools Console 中直接用 `fetch` 调用 API，注意带上 CSRF token。
- 不要在 oyd.jsx 中硬编码测试数据，用 `console.log(JSON.stringify(payload))` 打印实际 payload 后复制到 CLI 验证。
