---
name: tencent-cloud-cos
description: >
  腾讯云对象存储(COS)和数据万象(CI)集成技能。当用户需要上传、下载、管理云存储文件，
  或需要进行图片处理（质量评估、超分辨率、抠图、二维码识别、水印）、智能图片搜索、
  文档转PDF、视频智能封面生成等操作时使用此技能。
---

# 腾讯云 COS 技能

集成腾讯云对象存储(COS)和数据万象(CI)的完整能力。提供三种执行方式按优先级降级，确保操作始终可完成：

1. **方式一：cos-mcp MCP 工具**（优先） — 功能最全，支持存储操作 + 图片处理 + 智能搜索 + 文档媒体处理
2. **方式二：COS Node.js SDK 脚本** — 通过 `scripts/cos_node.mjs` 执行存储操作
3. **方式三：COSCMD 命令行工具** — 通过 shell 命令执行存储操作

## 执行策略

按以下顺序尝试，遇到失败自动降级到下一种方式：

```
cos-mcp MCP 工具可用？（先安装再检测）
  ├─ 是 → 使用方式一（全部功能）
  └─ 否 → Node.js 可用？（node --version）
              ├─ 是 → 安装 cos-nodejs-sdk-v5（若未安装）→ 使用方式二（存储操作）
              └─ 否 → coscmd 命令可用？（which coscmd）
                        ├─ 是 → 使用方式三（存储操作）
                        └─ 否 → 提示用户安装任一工具
```

**判断方式一是否可用**：
1. 先在当前项目安装 cos-mcp：`npm install cos-mcp`（若无 `package.json` 先 `npm init -y`）
2. 尝试调用 `getCosConfig` MCP 工具，若工具存在且返回结果则可用
3. 若 MCP 工具仍不可用（客户端未配置 MCP 服务器），则降级到方式二

**判断方式二是否可用**：执行 `node --version` 确认 Node.js 可用，然后检查 `cos-nodejs-sdk-v5` 是否已安装（`node -e "require('cos-nodejs-sdk-v5')"`），若未安装则先执行 `npm install cos-nodejs-sdk-v5` 安装后再使用。
**判断方式三是否可用**：执行 `which coscmd` 或 `coscmd --version` 有输出则可用。

## 环境配置

三种方式均需要以下腾讯云凭证：

| 参数 | 说明 | 示例 |
|------|------|------|
| SecretId | API 密钥 ID | `AKIDxxxxxxxx` |
| SecretKey | API 密钥 Key | `xxxxxxxx` |
| Region | 存储桶区域 | `ap-guangzhou` |
| Bucket | 存储桶名称 | `mybucket-1250000000` |
| DatasetName | 数据万象数据集名称（仅智能搜索需要） | `my-dataset` |

### 凭证持久化

各方式的凭证存储位置不同，**配置一次后重启会话无需重新设置**：

| 方式 | 持久化位置 | 说明 |
|------|-----------|------|
| 方式一 cos-mcp | 客户端 MCP 配置文件 | 写入后由客户端自动加载，无需重复配置 |
| 方式二 Node SDK | shell 配置文件（`~/.zshrc` 或 `~/.bashrc`） | 需将环境变量写入 shell 配置文件 |
| 方式三 COSCMD | `~/.cos.conf` | `coscmd config` 自动写入，后续直接可用 |

**方式二环境变量持久化**：首次配置时将凭证写入 shell 配置文件：

```bash
cat >> ~/.zshrc << 'EOF'
export TENCENT_COS_SECRET_ID="<替换为 API 密钥 ID>"
export TENCENT_COS_SECRET_KEY="<替换为 API 密钥 Key>"
export TENCENT_COS_REGION="<替换为存储桶区域>"
export TENCENT_COS_BUCKET="<替换为存储桶名称>"
EOF
source ~/.zshrc
```

若使用 bash，将 `~/.zshrc` 替换为 `~/.bashrc`。

> 密钥信息不要在对话中明文展示，引导用户自行编辑配置文件设置。

---

## 方式一：cos-mcp MCP 工具（优先）

> GitHub: https://github.com/Tencent/cos-mcp

### 安装配置

#### 1. 安装 cos-mcp 到当前项目

```bash
npm install cos-mcp
```

若当前目录无 `package.json`，先执行 `npm init -y`。

#### 2. 在客户端 MCP 配置中注册

在客户端（Claude Desktop、CodeBuddy 等）的 MCP 配置文件中注册：

```json
{
  "mcpServers": {
    "cos-mcp": {
      "command": "npx",
      "args": ["cos-mcp", "--connectType=stdio"],
      "env": {
        "TENCENT_COS_SECRET_ID": "<SecretId>",
        "TENCENT_COS_SECRET_KEY": "<SecretKey>",
        "TENCENT_COS_REGION": "<Region>",
        "TENCENT_COS_BUCKET": "<Bucket>"
      }
    }
  }
}
```

完整配置模板见 `references/config_template.json`。

### 存储操作工具

| 工具 | 用途 | 必需参数 | 可选参数 |
|------|------|----------|----------|
| `getCosConfig` | 获取当前配置信息 | 无 | 无 |
| `putObject` | 上传本地文件 | `filePath` | `fileName`, `targetDir` |
| `putString` | 上传字符串内容 | `content`, `fileName` | `targetDir`, `contentType` |
| `putBase64` | 上传 base64 内容 | `base64Content`, `fileName` | `targetDir`, `contentType` |
| `putBuffer` | 上传 buffer 内容 | `content`, `fileName` | `targetDir`, `contentType`, `encoding` |
| `putObjectSourceUrl` | 通过 URL 上传 | `sourceUrl` | `fileName`, `targetDir` |
| `getObject` | 下载文件 | `objectKey` | 无 |
| `getBucket` | 列出文件 | 无 | `Prefix` |
| `getObjectUrl` | 获取签名下载链接 | `objectKey` | 无 |

### 图片处理工具（数据万象 CI）

| 工具 | 用途 | 必需参数 | 可选参数 |
|------|------|----------|----------|
| `imageInfo` | 获取图片元信息 | `objectKey` | 无 |
| `assessQuality` | 图片质量评估 | `objectKey` | 无 |
| `aiSuperResolution` | AI 超分辨率 | `objectKey` | 无 |
| `aiPicMatting` | AI 智能抠图 | `objectKey` | `width`, `height` |
| `aiQrcode` | 二维码识别 | `objectKey` | 无 |
| `waterMarkFont` | 添加文字水印 | `objectKey` | `text` |

### 智能搜索工具

需要预先在数据万象控制台创建数据集。

| 工具 | 用途 | 必需参数 |
|------|------|----------|
| `imageSearchPic` | 以图搜图 | `uri` |
| `imageSearchText` | 文本搜图 | `text` |

### 文档与媒体处理工具

| 工具 | 用途 | 必需参数 |
|------|------|----------|
| `createDocToPdfJob` | 文档转 PDF | `objectKey` |
| `describeDocProcessJob` | 查询文档任务结果 | `jobId` |
| `createMediaSmartCoverJob` | 视频智能封面 | `objectKey` |
| `describeMediaJob` | 查询媒体任务结果 | `jobId` |

---

## 方式二：COS Node.js SDK 脚本

> 官方文档: https://www.tencentcloud.com/zh/document/product/436/8629

当 cos-mcp MCP 工具不可用时，使用 `scripts/cos_node.mjs` 脚本通过 `cos-nodejs-sdk-v5` SDK 执行存储操作。

### 安装依赖

使用方式二前需确保 `cos-nodejs-sdk-v5` 已安装。执行检查和安装：

```bash
# 检查是否已安装
node -e "require('cos-nodejs-sdk-v5')" 2>/dev/null || npm install cos-nodejs-sdk-v5
```

若在用户项目中操作，建议安装到项目目录；若无 `package.json`，先 `npm init -y` 再安装。

### 使用方式

通过 `node scripts/cos_node.mjs <action> [参数]` 执行，脚本从环境变量读取凭证。

#### 上传文件

```bash
node scripts/cos_node.mjs upload --file /path/to/local/file.jpg --key remote/path/file.jpg
```

#### 上传字符串内容

```bash
node scripts/cos_node.mjs put-string --content "文本内容" --key remote/path/file.txt --content-type "text/plain"
```

#### 下载文件

```bash
node scripts/cos_node.mjs download --key remote/path/file.jpg --output /path/to/save/file.jpg
```

#### 列出文件

```bash
node scripts/cos_node.mjs list --prefix "images/"
```

#### 获取签名 URL

```bash
node scripts/cos_node.mjs sign-url --key remote/path/file.jpg --expires 3600
```

#### 删除文件

```bash
node scripts/cos_node.mjs delete --key remote/path/file.jpg
```

#### 查看文件信息

```bash
node scripts/cos_node.mjs head --key remote/path/file.jpg
```

所有命令输出 JSON 格式结果，可通过 `$?` 判断成功（0）或失败（非0）。

### 限制

方式二仅支持存储操作（上传、下载、列表、删除、签名URL），**不支持**图片处理、智能搜索、文档转换等数据万象功能。

---

## 方式三：COSCMD 命令行工具

> 官方文档: https://www.tencentcloud.com/zh/document/product/436/10976

当方式一和方式二均不可用时，使用 COSCMD Python 命令行工具。

### 安装配置

```bash
pip install coscmd
coscmd config -a $TENCENT_COS_SECRET_ID -s $TENCENT_COS_SECRET_KEY -b $TENCENT_COS_BUCKET -r $TENCENT_COS_REGION
```

配置写入 `~/.cos.conf`，后续使用无需重复配置。

### 常用命令

#### 上传

```bash
# 上传单个文件
coscmd upload /path/to/file.jpg remote/path/file.jpg

# 递归上传目录
coscmd upload -r /path/to/folder/ remote/folder/
```

#### 下载

```bash
# 下载单个文件
coscmd download remote/path/file.jpg /path/to/save/file.jpg

# 递归下载目录
coscmd download -r remote/folder/ /path/to/save/
```

#### 列出文件

```bash
# 列出指定前缀下的文件
coscmd list images/

# 递归列出并统计
coscmd list -r images/
```

#### 删除

```bash
# 删除单个文件
coscmd delete remote/path/file.jpg

# 递归删除目录
coscmd delete -r remote/folder/ -f
```

#### 获取签名 URL

```bash
# 获取有效期 3600 秒的下载链接
coscmd signurl remote/path/file.jpg -t 3600
```

#### 查看文件信息

```bash
coscmd info remote/path/file.jpg
```

#### 复制/移动

```bash
# 桶内复制
coscmd copy <BucketName-APPID>.cos.<Region>.myqcloud.com/source.jpg dest.jpg

# 移动（复制后删除源文件）
coscmd move <BucketName-APPID>.cos.<Region>.myqcloud.com/source.jpg dest.jpg
```

### 限制

方式三仅支持存储操作，**不支持**图片处理、智能搜索、文档转换等数据万象功能。

---

## 功能对照表

| 功能 | 方式一 cos-mcp | 方式二 Node SDK | 方式三 COSCMD |
|------|:-:|:-:|:-:|
| 上传文件 | ✅ | ✅ | ✅ |
| 上传字符串/Base64 | ✅ | ✅ | ❌ |
| 通过 URL 上传 | ✅ | ❌ | ❌ |
| 下载文件 | ✅ | ✅ | ✅ |
| 列出文件 | ✅ | ✅ | ✅ |
| 获取签名 URL | ✅ | ✅ | ✅ |
| 删除文件 | ❌ | ✅ | ✅ |
| 查看文件信息 | ❌ | ✅ | ✅ |
| 递归上传/下载目录 | ❌ | ❌ | ✅ |
| 图片处理（CI） | ✅ | ❌ | ❌ |
| 智能搜索 | ✅ | ❌ | ❌ |
| 文档转 PDF | ✅ | ❌ | ❌ |
| 视频智能封面 | ✅ | ❌ | ❌ |

## 工作流程

### 文件上传

1. 按执行策略确定可用方式
2. 确认本地文件路径存在
3. 执行上传：
   - 方式一：调用 `putObject`，上传后调用 `getObjectUrl` 获取链接
   - 方式二：`node scripts/cos_node.mjs upload --file <path> --key <key>`，再 `sign-url` 获取链接
   - 方式三：`coscmd upload <localpath> <cospath>`，再 `coscmd signurl <cospath>` 获取链接
4. 返回访问链接给用户

### 文件下载

1. 按执行策略确定可用方式
2. 执行下载：
   - 方式一：调用 `getObject`
   - 方式二：`node scripts/cos_node.mjs download --key <key> --output <path>`
   - 方式三：`coscmd download <cospath> <localpath>`
3. 返回本地文件路径

### 图片处理（仅方式一）

1. 确认目标图片已在存储桶中（必要时先上传）
2. 调用对应图片处理工具
3. 若方式一不可用，提示用户：图片处理功能需要 cos-mcp MCP 工具支持

### 文档转换（仅方式一）

1. 确认文档已在存储桶中
2. 调用 `createDocToPdfJob` 创建任务
3. 使用 `describeDocProcessJob` 轮询结果
4. 若方式一不可用，提示用户：文档转换功能需要 cos-mcp MCP 工具支持

## 注意事项

- 所有文件路径（`objectKey`/`cospath`/`--key`）均为存储桶内的相对路径，如 `images/photo.jpg`
- 图片处理、智能搜索、文档转换等数据万象功能**仅方式一可用**，方式二和方式三仅覆盖基础存储操作
- 智能搜索需预先创建数据集并建立索引
- 文档转换和视频封面为异步任务，需通过 `jobId` 轮询结果
- 上传时如未指定目标文件名，默认使用原始文件名
- 方式二脚本源码见 `scripts/cos_node.mjs`
- MCP 工具详细参数参考见 `references/api_reference.md`
- MCP 配置模板见 `references/config_template.json`
