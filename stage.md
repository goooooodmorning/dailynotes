# 赛事报名与多阶段投稿 API 接口文档

## 1. 通用约定

* **Base URL**: `/api/v1`
* **认证方式**: 请求头需携带 Bearer Token：`Authorization: Bearer <access_token>`
* **数据格式**: 请求与响应均采用 `application/json; charset=utf-8`，时间字段统一采用 ISO 8601 格式（含时区）。
* **标准错误响应结构**:
  ```json
  {
    "detail": "当前非初赛报名开放时间"
  }
  ```

### 核心枚举字典

| 字段名 | 枚举值 | 说明 |
| :--- | :--- | :--- |
| `stage_status` | `PRE_LOCKED` | 初赛未开放（页面展示未开放缺省页） |
| | `PRE_OPEN` | 初赛报名与投稿进行中 |
| | `SEMI_LOCKED` | 复赛未开放 / 初赛评审间隙期（历史投稿只读） |
| | `SEMI_OPEN` | 复赛进行中（晋级者可补充复赛附件及修改初赛附件） |
| | `FINAL_LOCKED` | 决赛未开放 / 复赛评审间隙期（历史投稿只读） |
| | `FINAL_OPEN` | 决赛进行中（晋级者可补充决赛附件及修改历史附件） |
| | `ENDED` | 赛事已全部结束（全量只读） |
| `qualified_round` | `NONE` | 未报名 |
| | `PRE` | 初赛资格 |
| | `SEMI` | 晋级复赛 |
| | `FINAL` | 晋级决赛 |
| `audit_status` | `PENDING` | 待审核 |
| | `APPROVED` | 审核通过 |
| | `REJECTED` | 审核驳回 |
| `file_key` | `pre_doc` | 初赛方案文档 |
| | `semi_video` | 复赛演示视频 |
| | `final_ppt` | 决赛终审路演 PPT |

---

## 2. 接口详情

### 2.1 获取赛程全局状态与用户资格

前端进入活动主页面时调用，用于控制顶部 Tab 显隐、未开放缺省页、主操作按钮文案及倒计时校准。

* **请求方式**: `GET`
* **接口路径**: `/api/v1/competition/{contest_id}/state`

#### 路径参数

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `contest_id` | integer | 是 | 赛事 ID |

#### 响应示例 (`200 OK`)

```json
{
  "stage_status": "SEMI_OPEN",
  "server_time": "2026-12-05T14:20:00+08:00",
  "next_stage_time": "2026-12-15T23:59:59+08:00",
  "has_submitted": true,
  "qualified_round": "SEMI",
  "audit_status": "APPROVED"
}
```

#### 响应字段说明

| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `stage_status` | string | 当前全局赛程状态，见枚举字典 |
| `server_time` | string | 服务器当前时间，前端用于计算本地时钟偏移量校准倒计时 |
| `next_stage_time` | string \| null | 当前阶段截止时间（或下一阶段开启时间），`ENDED` 时为 `null` |
| `has_submitted` | boolean | 当前用户是否已完成初赛报名投稿 |
| `qualified_round` | string | 当前用户最高晋级轮次 |
| `audit_status` | string \| null | 当前投稿审核状态，未报名时为 `null` |

---

### 2.2 获取“我的投稿”详情及槽位权限

渲染“我的投稿”Tab 及“修改/补充投稿”弹窗时调用。后端已根据当前时间窗口与用户晋级资格计算好每个附件槽位的读写权限，前端直接按 `slot_permissions` 渲染即可。

* **请求方式**: `GET`
* **接口路径**: `/api/v1/submission/{contest_id}/me`

#### 路径参数

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `contest_id` | integer | 是 | 赛事 ID |

#### 响应示例 1：已报名且晋级复赛 (`200 OK`)

```json
{
  "submitted": true,
  "stage_status": "SEMI_OPEN",
  "qualified_round": "SEMI",
  "audit_status": "APPROVED",
  "audit_remark": null,
  "meta": {
    "work_name": "多模态智能巡检助手",
    "track_id": "TRACK_AI_AGENT",
    "team_info": {
      "team_name": "破晓小队",
      "leader": "张三"
    }
  },
  "files_data": {
    "pre_doc": {
      "name": "初赛技术方案_v1.pdf",
      "url": "[https://cdn.example.com/contest/pre_10293.pdf](https://cdn.example.com/contest/pre_10293.pdf)",
      "size": 4194304,
      "updated_at": "2026-11-22T18:30:00+08:00"
    }
  },
  "slot_permissions": {
    "pre_doc": {
      "visible": true,
      "editable": true,
      "required": true
    },
    "semi_video": {
      "visible": true,
      "editable": true,
      "required": true
    },
    "final_ppt": {
      "visible": false,
      "editable": false,
      "required": false
    }
  }
}
```

#### 响应示例 2：当前阶段未报名 (`200 OK`)

```json
{
  "submitted": false,
  "stage_status": "PRE_OPEN"
}
```

#### 错误响应

* `403 Forbidden`: `stage_status` 为 `PRE_LOCKED`（初赛尚未开放）时调用此接口。

---

### 2.3 初赛首次报名与投稿

仅在 `PRE_OPEN`（初赛开放期）允许调用，用于创建唯一投稿记录并锁定作品基础信息。

* **请求方式**: `POST`
* **接口路径**: `/api/v1/submission/create`

#### 请求体参数 (`application/json`)

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `contest_id` | integer | 是 | 赛事 ID |
| `work_name` | string | 是 | 作品名称（2~128 字符，**提交后永久锁定**） |
| `track_id` | string | 是 | 参赛赛道标识（**提交后永久锁定**） |
| `team_info` | object | 否 | 团队成员等扩展信息（JSON 对象，**提交后永久锁定**） |
| `files` | object | 是 | 附件映射表，首次提交必须包含 `pre_doc` 键 |
| `files.pre_doc.name` | string | 是 | 原始文件名 |
| `files.pre_doc.url` | string | 是 | 对象存储（OSS/COS）上传后的完整访问地址 |
| `files.pre_doc.size` | integer | 是 | 文件字节大小（Byte，需大于 0） |

#### 请求示例

```json
{
  "contest_id": 101,
  "work_name": "多模态智能巡检助手",
  "track_id": "TRACK_AI_AGENT",
  "team_info": {
    "team_name": "破晓小队",
    "leader": "张三"
  },
  "files": {
    "pre_doc": {
      "name": "初赛技术方案_v1.pdf",
      "url": "[https://cdn.example.com/contest/pre_10293.pdf](https://cdn.example.com/contest/pre_10293.pdf)",
      "size": 4194304
    }
  }
}
```

#### 响应示例 (`200 OK`)

```json
{
  "submission_id": 8842,
  "message": "报名投稿成功"
}
```

#### 错误响应

* `400 Bad Request`: 非 `PRE_OPEN` 时间窗口，或该用户已提交过报名。
* `422 Unprocessable Entity`: 缺少必填的 `pre_doc` 文件或字段格式校验失败。

---

### 2.4 增量补充与替换附件

用于初赛、复赛、决赛开放期修改已有附件或补充当前轮次的新附件。**接口只接受文件字段**，不接受任何基础元数据修改；提交时需传入当前阶段必填附件及本次变更的附件。

* **请求方式**: `PATCH`
* **接口路径**: `/api/v1/submission/files`

#### 请求体参数 (`application/json`)

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `contest_id` | integer | 是 | 赛事 ID |
| `files` | object | 是 | 需更新或补充的附件字典，Key 必须为合法的 `file_key`（如 `pre_doc`、`semi_video`、`final_ppt`） |
| `files.<key>.name` | string | 是 | 文件名 |
| `files.<key>.url` | string | 是 | 新文件的 OSS/COS 访问地址 |
| `files.<key>.size` | integer | 是 | 文件字节大小（Byte） |

#### 请求示例（复赛期间补充演示视频，并同步替换初赛方案文档）

```json
{
  "contest_id": 101,
  "files": {
    "pre_doc": {
      "name": "初赛技术方案_修订版.pdf",
      "url": "[https://cdn.example.com/contest/pre_10999.pdf](https://cdn.example.com/contest/pre_10999.pdf)",
      "size": 5120000
    },
    "semi_video": {
      "name": "复赛系统实机演示.mp4",
      "url": "[https://cdn.example.com/contest/semi_20481.mp4](https://cdn.example.com/contest/semi_20481.mp4)",
      "size": 52428800
    }
  }
}
```

#### 响应示例 (`200 OK`)

```json
{
  "message": "投稿材料已更新",
  "files_data": {
    "pre_doc": {
      "name": "初赛技术方案_修订版.pdf",
      "url": "[https://cdn.example.com/contest/pre_10999.pdf](https://cdn.example.com/contest/pre_10999.pdf)",
      "size": 5120000,
      "updated_at": "2026-12-06T10:12:30+08:00"
    },
    "semi_video": {
      "name": "复赛系统实机演示.mp4",
      "url": "[https://cdn.example.com/contest/semi_20481.mp4](https://cdn.example.com/contest/semi_20481.mp4)",
      "size": 52428800,
      "updated_at": "2026-12-06T10:12:30+08:00"
    }
  }
}
```

#### 错误响应

* `403 Forbidden`: 当前处于锁定间隙期（如 `SEMI_LOCKED`）、用户未晋级当前阶段，或尝试修改无权限的 `file_key`。
* `404 Not Found`: 未找到用户的初始报名记录。
* `422 Unprocessable Entity`: 缺少当前阶段必填的补充材料（例如在 `SEMI_OPEN` 期间首次补充材料时未包含 `semi_video`）。

---

## 3. 错误码与前端交互建议

| HTTP 状态码 | 触发场景 | 前端交互处理建议 |
| :--- | :--- | :--- |
| `200` | 请求成功 | 正常渲染数据或弹出成功 Toast，刷新页面状态 |
| `400` | 重复报名、时间窗口已过 | 弹出提示并重新调用 `GET /state` 刷新页面阶段状态 |
| `403` | 未开放期访问、越权修改附件 | 隐藏编辑按钮，切换为只读视图并提示“当前阶段暂未开放编辑” |
| `422` | 缺少必填附件或参数校验不通过 | 高亮对应未上传的附件槽位，提示补全材料 |
