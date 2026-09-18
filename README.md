# docmost-build

自定义 Docmost 镜像构建仓库：GitHub Actions 检出官方源码 -> 应用 `patches/` 下的补丁 -> 构建镜像并推送到 Docker Hub。

## 目录结构

- `.github/workflows/build-docmost-fiexed.yml`：唯一的工作流，手动触发
- `patches/docmost-docx-export.patch`：DOCX 导出补丁（含 ZIP 附件路径修复）

## 前置配置

仓库 Settings 中需要：

- Variables -> `DOCKERHUB_USERNAME`：Docker Hub 用户名（必须是 Variable，工作流用 `vars.DOCKERHUB_USERNAME` 读取）
- Secrets -> `DOCKERHUB_TOKEN`：Docker Hub Access Token，权限 Read & Write

## 触发构建

Actions -> Build Docmost Fixed (Docker Hub) -> Run workflow：

- `version`：Docmost 源码 tag，默认 `v0.96.0`
- `tag_suffix`：镜像 tag 前缀，默认 `fixed`

产物：`docker.io/<用户名>/docmost:<tag_suffix>-<version>`，例如 `liwella/docmost:fixed-v0.96.0`。

## 补丁内容

`patches/docmost-docx-export.patch` 基于 `v0.96.0` 生成，包含：

- 新增 DOCX 导出实现（`apps/server/src/integrations/export/docx-export.service.ts`）与 `POST /api/docx-export` 接口，前端导出弹窗去掉企业版限制
- Word 导出改为产出 zip：`<页面标题>.zip` 里是 `<页面标题>.docx` 加同级 `files/{附件id}/{ASCII 文件名}`
- 页面标题导出成 docx 正文里的第一个一级标题（标题在 Docmost 里是独立字段、不写在正文内容里，之前整段丢失）
- HTML 导出补上表格样式（`1px solid #ced4da` 边框、表头 `#f1f3f5` 底色加粗、单元格内边距），导出页面原本不带任何样式表，表格在浏览器里是无边框的；并在 head 里声明 `<meta charset="utf-8" />`，避免本地打开时中文乱码
- docx 正文里的图片直接嵌入（SVG 先用 resvg 光栅化成 PNG）；附件/PDF/视频/音频写成超链接，目标是包内相对路径 `files/...`，解压后点击即打开本地文件，全程不访问 Docmost
- 代码块导出为等宽字体（Consolas 9pt）+ 灰底 + 细边框，逐行输出并保留缩进（Word 会忽略文本里的原始换行，所以按行拆成多个 run）
- mermaid 代码块导出成图片：前端用 mermaid 渲染 SVG（`htmlLabels: false`，容器的 resvg 画不了 `<foreignObject>`），服务端 resvg 光栅化成 PNG 嵌进 docx；渲染失败的图仍按代码文本导出
- 表格还原编辑器样式：1px `#ced4da` 边框、表头 `#F1F3F5` 灰底加粗且跨页重复、单元格 3px/5px 内边距、列宽按编辑器拖拽的 `colwidth` 等比缩放（没拖过就等分），整表固定布局占满正文宽度
- 过高的图自动限高，避免超出页面可用高度被 Word 截断
- 修复 ZIP 导出中附件路径多一个前导 `/` 的问题
- Dockerfile 增加中文字体 `fonts-wqy-microhei`，否则 Word 中中文渲染为空白

## 附件是怎么取的

- **Word/DOCX 导出**（本补丁新增）：导出的是 `<页面标题>.zip`，解压得到 `<页面标题>.docx` 和同级的 `files/` 目录。图片内容嵌在 docx 里；其他文件以超链接出现在正文，链接是相对路径，点击直接打开解压出来的本地文件，不访问 Docmost。
  - docx 必须和 `files/` 目录放在一起；单独把 docx 发出去，附件链接会失效。
  - 包里附件用 ASCII 文件名：`31098需求说明书.pdf` -> `31098.pdf`，`meeting 记录.m4a` -> `meeting.m4a`，纯中文名退化为附件 id 前 8 位（`演示视频.mp4` -> `44444444.mp4`）。Word 打不开带中文或 `%XX` 转义的超链接目标，所以正文里的链接文字保留原始文件名，链接指向的是 ASCII 文件。
  - 兜底：某个附件读不到（文件缺失或无权访问）时，该处链接退回在线地址 `<APP_URL>/api/files/{附件id}/{文件名}`。
- **ZIP 导出**（Docmost 原生，Markdown/HTML）：附件本体在压缩包里（`files/{附件id}/{文件名}`，保留原始文件名），页面里对附件的引用会被改写成相对路径 `files/...`，同样离线可用。

## mermaid 图是怎么导出的

- 图在**浏览器**里渲染：点导出时前端把页面里的 mermaid 代码块渲染成 SVG，随导出请求一起发给服务端，服务端再光栅化成 PNG 嵌进 docx，所以版式和你编辑器里看到的一致。
- 用 API/脚本直接调 `POST /api/docx-export`（不经过页面按钮）时没有 SVG，mermaid 代码块会按代码文本导出。
- 容器要有中文字体（Dockerfile 已装 `fonts-wqy-microhei`、`fonts-dejavu-core`），否则图里的中文会变成空白。
- 一次导出携带的 SVG 总量限制在 400KB 以内（这是为了不超过请求体 1MB 的限制）；超出部分仍按代码文本导出。

## 升级 Docmost 版本

修改 workflow 的 `version` 输入后重新构建即可。若构建日志里 `git apply` 报冲突，需要按新版源码重新生成补丁；最容易冲突的文件是 `apps/client/src/components/common/export-modal.tsx`。

## 服务器侧注意

- 导出内容本身不再依赖 `APP_URL`；只有上面那条兜底链接会用到，仍建议配成真实访问地址
- 改完补丁后，解压导出的 zip、用 Word 打开 docx，点一个附件链接确认能打开本地文件
