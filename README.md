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
- 导出 Word 时内嵌图片（SVG 用 resvg 光栅化），附件/视频等以超链接形式写入正文，链接指向 `/api/files/{id}/{name}`
- 修复 ZIP 导出中附件路径多一个前导 `/` 的问题
- Dockerfile 增加中文字体 `fonts-wqy-microhei`，否则 Word 中中文渲染为空白

## 升级 Docmost 版本

修改 workflow 的 `version` 输入后重新构建即可。若构建日志里 `git apply` 报冲突，需要按新版源码重新生成补丁；最容易冲突的文件是 `apps/client/src/components/common/export-modal.tsx`。

## 服务器侧注意

- 容器环境变量 `APP_URL` 必须是真实访问地址，否则 Word 里的附件超链接会指向 localhost
- 点击 Word 中的附件链接需要浏览器已登录 Docmost
