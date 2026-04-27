# Hugo 站点搭建与发布说明

本文档总结本仓库（PaperMod 主题 + GitHub Pages）的 **本地搭建** 与 **自动发布** 流程，便于复现与交接。

---

## 1. 技术栈与产物

| 项 | 说明 |
| --- | --- |
| 静态站点生成器 | [Hugo](https://gohugo.io/)（CI 使用 **Extended** 版本，以支持主题等资源管道） |
| 主题 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod)（以 **Git 子模块** 形式位于 `themes/PaperMod`） |
| 配置文件 | 站点根目录 `hugo.toml` |
| 构建输出 | `public/`（由 `hugo` 生成，**勿**将构建产物当作唯一信源提交；可按需加入 `.gitignore`） |
| 发布目标 | **GitHub Pages**（通过 Actions 上传 `public` 目录） |

---

## 2. 前置条件

- 已安装 **Hugo Extended**（本地版本建议与线上接近；当前 CI 使用 `latest`）。
- 已安装 **Git**，且能访问 GitHub。
- 克隆仓库时若主题未出现，需拉取子模块（见下文）。

---

## 3. 获取代码与子模块

主题通过子模块引用，见仓库根目录 `.gitmodules`：

```text
themes/PaperMod → https://github.com/adityatelange/hugo-PaperMod.git
```

**首次克隆：**

```bash
git clone <你的仓库 URL>
cd <仓库目录>
git submodule update --init --recursive
```

**已有仓库、子模块未初始化：**

```bash
git submodule update --init --recursive
```

> CI 已在 `actions/checkout` 中配置 `submodules: recursive`，推送后构建机会自动拉取主题。

---

## 4. 本地开发与预览

在仓库根目录执行：

```bash
hugo server -D
```

常用参数说明：

| 参数 | 用途 |
| --- | --- |
| `-D` | 包含草稿（draft）内容 |
| `--bind 0.0.0.0` | 局域网内其它设备访问 |
| `--port 1313` | 指定端口（默认 1313） |
| `--baseURL http://localhost:1313/` | 本地绝对链接与资源路径与访问地址一致 |

**部署在子路径时（本仓库 `baseURL` 含 `/hugo_mork_data/`）：**  
若需本地完全模拟线上资源路径，启动时使用与访问浏览器一致的 `baseURL`，例如：

```bash
hugo server -D --baseURL "http://localhost:1313/hugo_mork_data/" --appendPort=false
```

（具体端口、路径以你实际访问为准；不一致时易出现样式或脚本路径偏差。）

生成静态文件到 `public/`：

```bash
hugo --minify
```

> 线上 CI 使用 **`hugo --minify`**。内联脚本与样式在压缩后更易暴露命名冲突等问题；自定义脚本建议避免过短的全局变量名，并与主题全局 CSS（如 PaperMod 对 `table` 的重置）做作用域隔离。

---

## 5. 站点配置要点（`hugo.toml`）

与本发布流程强相关的配置包括：

- **`baseURL`**：须与 **GitHub Pages 最终访问地址** 一致（含协议、域名、大小写习惯、**仓库子路径**）。例如项目站为 `https://<user>.github.io/<repo>/`。
- **`theme = 'PaperMod'`**：与子模块目录名一致。
- **`languageCode` / `title` / `[params]`**：站点元信息与主题参数；按需维护。
- **`[params.assets]`**：自定义 favicon 等静态资源文件名时，需保证 `static/` 下存在对应文件，避免 404。

更多主题选项见 [PaperMod 文档](https://github.com/adityatelange/hugo-PaperMod/wiki)。

---

## 6. 目录结构（与本站相关）

| 路径 | 说明 |
| --- | --- |
| `content/` | 文章与页面 Markdown |
| `layouts/` | 覆盖主题的模板（如 `_default/list.html`、`partials/`） |
| `static/` | 原样复制到站点根路径的静态文件（如 `favicon.svg`、`vendor/`） |
| `assets/css/extended/` | PaperMod 会合并的扩展样式 |
| `data/` | 数据文件（如节假日 JSON 等），模板内可读 |
| `themes/PaperMod/` | 子模块主题，一般不在主仓库直接改主题源码 |

---

## 7. 发布流程（GitHub Actions）

工作流文件：`.github/workflows/deploy.yml`。

**触发条件：**

- 推送到分支 **`main`**
- 或手动 **`workflow_dispatch`**

**构建任务概要：**

1. `actions/checkout@v4`，**递归子模块**，`fetch-depth: 0`（便于按 Git 信息生成内容时完整历史）。
2. `peaceiris/actions-hugo@v3`：`hugo-version: latest`，**`extended: true`**。
3. 执行 **`hugo --minify`**。
4. `actions/upload-pages-artifact@v3`：上传 **`./public`**。

**部署任务：**

- `actions/deploy-pages@v4`，环境为 **`github-pages`**。

**仓库设置（需在 GitHub 上完成一次）：**

1. **Settings → Pages**：Source 选择 **GitHub Actions**（而非旧版 `gh-pages` 分支直接托管）。
2. **Settings → Actions → General**：Pages 相关权限允许工作流写入（本 workflow 已声明 `pages: write`、`id-token: write`）。

推送 `main` 成功后，在 Actions 与 Pages 设置中可查看部署状态与站点 URL。

---

## 8. 常见问题

| 现象 | 可能原因 | 处理方向 |
| --- | --- | --- |
| 构建报找不到主题 | 子模块未拉取 | 本地执行 `git submodule update --init --recursive`；确认 CI `checkout` 含 `submodules: recursive` |
| 线上样式/脚本路径错误 | `baseURL` 与真实 Pages URL 不一致 | 修改 `hugo.toml` 的 `baseURL` 后重新构建发布 |
| 本地连接被拒绝 | 未启动 `hugo server` 或端口错误 | 确认命令与浏览器访问的 host/port 一致 |
| `hugo --minify` 后功能异常 | 压缩导致脚本冲突或语法敏感 | 调整内联脚本命名空间、加载顺序；避免与第三方全局名碰撞 |
| 日历等组件布局异常 | 主题全局 CSS 与组件假设冲突 | 在组件根容器上覆盖样式（例如恢复 `table` 的表格布局） |

---

## 9. 待确认 / 可自行补充

- 若将 **`public/`** 纳入版本控制，需团队约定是否与 CI 产物重复；多数做法为 **仅 CI 构建、不提交 public**。
- 固定 CI 的 Hugo 版本号（而非 `latest`）可提高构建可复现性，可按需在 `peaceiris/actions-hugo` 中改为具体版本号。

---

## 10. 参考链接

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [PaperMod Wiki](https://github.com/adityatelange/hugo-PaperMod/wiki)
- [peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo)
- [GitHub Pages 文档](https://docs.github.com/pages)
