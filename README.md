# 用户管理后台 CMS — 前端说明

GitHub Pages 静态前端，配合腾讯云 SCF Web 函数实现用户删除和额度发放。

---

## 部署原理

```
本地 push → GitHub Actions (deploy.yml)
               ↓
         注入 SCF_BASE_URL secret → index.html
               ↓
         发布到 gh-pages 分支 → GitHub Pages
```

`index.html` 里的 `__SCF_BASE_URL__` 是占位符，由 Actions 在 CI 阶段用 `sed` 替换为真实的 SCF 触发器地址。**本地直接打开 index.html 不可用**，必须通过 GitHub Pages 访问。

---

## 必要配置：GitHub Secrets

在仓库 **Settings → Secrets and variables → Actions → New repository secret** 添加：

| Secret 名称 | 值 | 说明 |
|---|---|---|
| `SCF_BASE_URL` | `https://service-xxxx.sh.apigw.tencentcs.com/release` | SCF 函数触发器 URL，**末尾不加斜杠** |

Secret 未配置或为空时，Actions 会将占位符替换为空字符串，导致请求打到 GitHub Pages 本身，返回 **HTTP 405**。

---

## 页面结构

```
index.html（单文件，含 CSS + HTML + JS）
│
├── 视图1：索引页（#view-index）
│   ├── 功能菜单：额度发放 / 用户删除
│   └── 底部密钥抽屉（滑出动画）
│
├── 视图2：额度发放（#view-grant）
│   ├── 查看当前资产（调用 /queryAsset）
│   ├── 选择业务线 / 额度类型
│   └── 发放（调用 /grantCredit）
│
└── 视图3：用户删除（#view-delete）
    ├── 输入手机号
    ├── 确认弹窗
    └── 删除（调用 /deleteUser）
```

---

## 认证流程

1. 首页选择功能后，底部抽屉滑出，要求输入 `ADMIN_SECRET`
2. 点击「验证并进入」时，前端用虚拟手机号 `__auth_check__` 调用 `/queryAsset` 探测密钥是否正确
   - 返回 `401` → 密钥错误，拒绝进入
   - 返回 `404` → 密钥正确（虚拟号码查不到用户，属正常），进入功能页
   - 返回其他非 2xx → 显示错误（含 HTTP 状态码）
3. 通过验证后，密钥存入 `window._adminSecret`，后续每次 API 请求自动携带

---

## API 调用说明

所有请求均为 `POST`，`Content-Type: application/json`，body 自动附带 `secret` 字段。

| 路径 | 功能 | 关键入参 | 关键出参 |
|---|---|---|---|
| `POST /queryAsset` | 查询用户资产 | `phoneNumber` | `userInfo`, `assetSummary`, `assetDetails` |
| `POST /grantCredit` | 发放额度 | `phoneNumber`, `amount`, `business_line`, `quota_type` | `success`, `message` |
| `POST /deleteUser` | 删除用户 | `phoneNumber` | `success`, `data.deleteAuth`, `data.deleteRecord` |

`business_line` 可选值：`einvoice`（发票重命名）、`creditid`（企业执照）

`quota_type` 可选值：`permanent_rename`（永久）、`temporary_rename`（临时）

---

## 主要 JS 函数

| 函数 | 说明 |
|---|---|
| `selectFeature(name)` | 高亮菜单项，打开底部密钥抽屉 |
| `handleDrawerConfirm()` | 验证密钥，通过后跳转对应功能页 |
| `cancelSelection()` | 关闭抽屉，重置菜单状态 |
| `apiFetch(path, data)` | 统一 fetch 封装，自动附带 secret |
| `queryAsset()` | 查询并渲染资产卡片 |
| `grantCredit()` | 发放额度，成功后自动刷新资产 |
| `confirmDelete()` | 弹出删除确认弹窗 |
| `doDelete()` | 确认后执行删除请求 |
| `showToast(msg, type)` | 顶部短暂提示（ok / err） |
| `setLoading(btn, loading)` | 按钮 loading 状态切换（含 spinner） |

---

## 本地调试

直接用浏览器打开 `index.html` 时，`SCF_BASE_URL` 仍是占位符字符串，请求会失败。如需本地调试，手动将 `index.html` 第 633 行改为真实 URL：

```js
// 临时本地调试，不要 commit 这行
const SCF_BASE_URL = 'https://your-real-scf-url.com';
```

或使用项目根目录的 `mock_test.py` 直接对 SCF 接口进行测试（Python 不受浏览器 CORS 限制）。

---

## 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| 点击验证显示「HTTP 405」 | `SCF_BASE_URL` secret 未配置或为空 | 在 GitHub Secrets 里添加正确的 SCF 触发器 URL |
| 点击验证显示「HTTP 405」 | SCF 函数 URL 未开启公网访问 | 腾讯云控制台 → 函数 URL → 启用公网访问 |
| 点击验证显示网络错误 | CORS 未配置 | 检查 SCF `ALLOWED_ORIGIN` 环境变量是否设为你的 GitHub Pages 域名 |
| 密钥正确但显示「密钥错误」 | `ADMIN_SECRET` 环境变量与前端输入不一致 | 检查 SCF 控制台环境变量 |
| Actions 运行成功但页面还是旧的 | gh-pages 缓存 | 强刷浏览器（Ctrl+Shift+R）或等 CDN 刷新（约 1 分钟） |
