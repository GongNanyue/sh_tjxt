# tj-media 媒资服务详解

## 一、服务概述

tj-media 是天机学堂的媒资管理服务，负责文件上传/管理和视频媒资的上传/播放/管理。该服务采用**策略模式**支持多云存储平台（阿里云OSS、腾讯云COS/VOD），并提供视频播放授权签名等功能。

**启动类：** `com.tianji.media.MediaApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-api | 公共 API 模块 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| spring-boot-starter-web | Web 框架 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| tencentcloud-sdk-java + cos_api + vod_api | 腾讯云SDK（COS对象存储 + VOD视频点播） |
| aliyun-sdk-oss | 阿里云OSS对象存储SDK |

## 三、核心业务逻辑

### 3.1 文件管理（FileController + FileServiceImpl）
- **上传文件**：接收 MultipartFile -> 调用云存储上传 -> 记录文件信息到数据库 -> 返回文件信息
- **获取文件信息**：根据ID查询文件元数据
- **删除文件**：删除数据库记录

### 3.2 视频媒资管理（MediaController + MediaServiceImpl）
- **分页搜索媒资**：管理端搜索已上传的媒资信息
- **保存媒资信息**：视频上传完成后保存媒资元数据
- **获取上传签名**：生成视频上传授权签名（客户端直传模式）
- **获取播放签名**：根据小节ID获取视频播放授权签名（先通过课程服务查小节关联的媒资ID）
- **获取预览签名**：管理端直接根据媒资ID获取播放授权签名
- **删除媒资**：支持单个和批量删除

### 3.3 云存储策略

通过 `IFileStorage` 和 `IMediaStorage` 接口抽象文件和视频的存储操作：

| 实现类 | 平台 | 用途 |
|--------|------|------|
| AliFileStorage | 阿里云OSS | 文件（图片等）存储 |
| TencentFileStorage | 腾讯云COS | 文件存储 |
| TencentMediaStorage | 腾讯云VOD | 视频媒资存储与播放 |

### 3.4 定时任务（PullEventTask）
拉取腾讯云VOD的事件通知（如转码完成、审核结果等），更新媒资状态。

## 四、数据模型

### 4.1 Media（媒资表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| fileId | String | 云端唯一标识 |
| filename | String | 文件名称 |
| mediaUrl | String | 媒体播放地址 |
| coverUrl | String | 媒体封面地址 |
| duration | Float | 视频时长（秒） |
| status | FileStatus | 状态：1上传中、2已上传 |
| size | Long | 媒资大小（字节） |

### 4.2 File（文件表）
普通文件（图片等）的元数据表。

## 五、API接口清单

### 文件接口（/files）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /files | 上传文件 |
| GET | /files/{id} | 获取文件信息 |
| DELETE | /files/{id} | 删除文件 |

### 媒资接口（/medias）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /medias | 分页搜索媒资信息 |
| POST | /medias | 上传视频后保存媒资信息 |
| GET | /medias/signature/upload | 获取视频上传授权签名 |
| GET | /medias/signature/play | 获取播放视频授权签名（按小节ID） |
| GET | /medias/signature/preview | 管理端获取预览视频授权签名（按媒资ID） |
| DELETE | /medias/{mediaId} | 删除媒资视频 |
| DELETE | /medias?ids= | 批量删除媒资视频 |

## 六、技术方案

1. **策略模式多云存储**：通过 `IFileStorage` / `IMediaStorage` 接口和 `PlatformProperties` 配置，运行时根据配置选择使用哪个云存储平台，实现存储服务的可插拔。
2. **客户端直传**：视频上传采用客户端直传模式 -- 后端生成上传签名，前端直接上传到云存储平台，减少后端带宽压力。
3. **播放授权签名**：视频播放时后端生成带有效期的播放签名，防止视频链接被盗用。
4. **事件拉取机制**：定时拉取云端事件通知（转码完成等），更新本地媒资状态。
