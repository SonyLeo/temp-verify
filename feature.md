## 文档一：功能设计文档 - 会话分享系统 V2.0

### **1. 项目概述**

#### **1.1 系统愿景**
构建一个健壮、高效且可扩展的会话分享后端服务。该系统不仅提供基础的会话分享与预览功能，还通过“智能链接”机制，无缝整合用户回流与拉新链路，最终成为一个可管理、可分析的用户增长工具。

#### **1.2 核心功能**
*   **创建分享**：用户可将一段对话生成一个唯一的、可通过 URL 访问的公开链接。
*   **预览分享**：任何拥有链接的人都可以查看分享的对话内容，并支持**静态**和**流式（打字机效果）**两种预览模式。
*   **智能路由**：分享链接能识别访问者身份。如果是分享创建者本人，则自动重定向回应用内继续对话，实现用户回流。
*   **分享管理**：用户可以查看自己创建的所有分享列表，并进行删除操作。

### **2. 核心流程与数据流**

#### **2.1 交互流程：创建分享**
1.  **用户操作**：用户在应用内选中一段对话，点击“分享”按钮。
2.  **前端请求**：前端应用收集选中的 `messages` 数组和 `originalConversationId`，通过 `POST /api/v1/shares` 接口发送给后端。
3.  **后端处理（智能幂等）**:  
    a.  服务接收请求，从认证信息中获取 `userId`。  
    b.  **计算内容哈希**：将 `messages` 数组的内容序列化并计算 SHA-256 哈希值 (`content_hash`)，以此作为这组分享内容的唯一指纹。  
    c.  **数据库查询**：使用 `(userId, originalConversationId, content_hash)` 查询 `shared_conversations` 表，检查是否存在完全相同的、未删除的分享。  
    d.  **逻辑分支**:  
        *   **若存在**：直接返回已存在的分享记录的 `shareId` 和 `shareUrl` (HTTP `200 OK`)。  
        *   **若不存在**：  
            i.  生成一个新的、唯一的 `shareId` (使用 nanoid)。  
            ii. **开启事务**：将分享元数据（`shareId`, `userId`, `content_hash` 等）写入 `shared_conversations` 表，并将每一条消息及其顺序写入   `shared_messages` 表。
            iii. **提交事务**。
            iv. 返回新生成的 `shareId` 和 `shareUrl` (HTTP `201 Created`)。
4.  **前端响应**：前端收到 `shareUrl` 后，将其展示给用户，用户可以复制和分享。

#### **2.2 交互流程：访问分享链接**
1.  **用户操作**：用户（访客或登录用户）在浏览器中打开链接 `https://yourdomain.com/share/{shareId}?mode=stream`。
2.  **后端智能路由 (`GET /share/:shareId`)**:  
    a.  后端接收请求，通过 `shareId` 查询 `shared_conversations` 表。若不存在或已软删除，则返回 404 页面。  
    b.  **异步更新浏览量**：在后台原子性地为该分享的 `view_count` 加 1。  
    c.  **身份识别**：检查请求中是否包含有效的认证信息 (`auth.optional`)。  
    d.  **逻辑分支**:  
        *   **访问者是创建者**：后端返回 HTTP `302` 重定向响应，目标地址为 `https://app.yourdomain.com/chat/{original_conversation_id}`。浏览器自动跳转。  
        *   **访问者是其他用户或访客**：后端返回预览页面的 内容 骨架。
3.  **前端渲染**:
    a.  页面加载后，前端 JavaScript 执行。  
    b.  **读取模式**：从 URL 中解析出 `mode=stream` 查询参数。  
    c.  **获取数据**：调用 `GET /api/v1/shares/{shareId}` 接口，从后端获取完整的消息列表。  
    d.  **条件渲染**：根据 `mode` 参数的值，选择渲染“打字机回放组件”或“静态内容展示组件”，将获取到的消息数据传入组件进行渲染。

#### **2.3 数据流向图**

<img width="1410" height="1302" alt="screenshot-2025-11-06T09-03-28" src="https://github.com/user-attachments/assets/fb3b930a-2f0b-4f8a-819e-025f01acc27a" />


### **3. 数据库设计 (MySQL)**

#### **3.1 Table: `shared_conversations` - 分享元数据表**
存储每一次分享的核心信息和统计数据。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT UNSIGNED` | PK, AI | 主键，唯一标识。 |
| `share_id` | `VARCHAR(20)` | NOT NULL, UNIQUE | **业务主键**。对外暴露的、URL友好的短ID，例如由nanoid生成。 |
| `original_conversation_id` | `VARCHAR(255)` | NOT NULL | 原始对话的ID，用于用户回流时的重定向。 |
| `created_by_id` | `BIGINT UNSIGNED` | NOT NULL, FK(`users.id`) | 创建该分享的用户的ID，外键关联到用户表。 |
| `first_message_preview` | `VARCHAR(255)` | NULL | **性能优化字段**。冗余存储分享内容的第一句话摘要，用于在“我的分享”列表中快速展示，避免查询 `shared_messages` 表。 |
| `content_hash` | `VARCHAR(64)` | NOT NULL | **幂等性关键字段**。由分享的所有消息内容计算出的SHA-256哈希值，用于判断两次分享的内容是否完全一致。 |
| `view_count` | `INT UNSIGNED` | NOT NULL, DEFAULT 0 | 浏览量计数器。 |
| `expires_at` | `DATETIME` | NULL | 分享的过期时间。`NULL` 表示永不过期。 |
| `is_deleted` | `TINYINT(1)` | NOT NULL, DEFAULT 0 | **软删除标志**。`0` 表示有效，`1` 表示已删除。 |
| `created_at` | `DATETIME` | NOT NULL, DEFAULT NOW() | 记录创建时间。 |
| `updated_at` | `DATETIME` | NOT NULL, ON UPDATE NOW() | 记录最后更新时间。 |

**索引设计**:
*   `uk_share_id` on `share_id`: 保证对外ID的唯一性，并为访问提供快速查询。
*   `idx_user_conversation_hash` on `(created_by_id, original_conversation_id, content_hash)`: **复合索引**，为创建分享时的幂等性检查提供极高的查询效率。
*   `idx_is_deleted` on `is_deleted`: 快速过滤已删除的记录。

#### **3.2 Table: `shared_messages` - 分享消息表**
存储分享中包含的具体消息内容，与 `shared_conversations` 表为一对多关系。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT UNSIGNED` | PK, AI | 主键。 |
| `shared_conversation_id` | `BIGINT UNSIGNED` | NOT NULL, FK(`shared_conversations.id`) | 外键，关联到对应的分享元数据记录。 |
| `is_user` | `TINYINT(1)` | NOT NULL | 消息发送方标识。`1` 表示用户，`0` 表示 AI。 |
| `content` | `TEXT` | NOT NULL | 消息的具体内容。使用 `TEXT` 类型以支持长对话。 |
| `order_index` | `SMALLINT UNSIGNED` | NOT NULL | **顺序保证字段**。消息在对话中的顺序，从0开始。查询时需按此字段排序。 |

**索引设计**:
*   `idx_shared_conversation_id` on `shared_conversation_id`: 通过分享ID快速查找所有相关消息。

### **4. 待讨论与优化的内容**
此部分内容用于团队评审和答辩，展示我们对系统未来演进的思考。

1.  **长对话的性能问题**：当前设计将所有消息一次性返回给前端。对于包含成百上千条消息的超长对话，可能会导致API响应慢和前端渲染压力大。
    *   **优化方向**：为 `GET /api/v1/shares/:shareId` 接口引入基于游标或页码的**消息分页机制**。
2.  **缓存策略**：对于热点分享（高 `view_count`），每次访问都查询数据库会造成不必要的压力。
    *   **优化方向**：引入 **Redis** 作为缓存层。当通过 `shareId` 查询时，首先检查 Redis。缓存内容可包括分享元数据和完整的消息列表。缓存的失效策略可以是 TTL（如1小时）或在分享被删除/更新时主动清除。
---

## 文档二：接口设计文档 (API Specification) - V2.0

### **1. 通用约定**

*   **Base URL**: `/api/v1`
*   **认证**: 凡是需要认证的接口，均需在 HTTP Header 中提供 `Authorization: Bearer <YOUR_JWT_TOKEN>`。
*   **成功响应体**:
    ```json
    { "data": { ... } }
    ```
*   **错误响应体**:
    ```json
    {
      "error": {
        "code": "ERROR_CODE_STRING",
        "message": "A human-readable error message."
      }
    }
    ```
*   **通用错误码**:
    *   `400 Bad Request` - `VALIDATION_ERROR`: 请求体或参数校验失败。
    *   `401 Unauthorized` - `UNAUTHENTICATED`: 未提供有效认证信息。
    *   `403 Forbidden` - `FORBIDDEN`: 权限不足。
    *   `404 Not Found` - `NOT_FOUND`: 资源不存在。
    *   `500 Internal Server Error` - `SERVER_ERROR`: 服务器内部错误。

### **2. 接口列表**

---

#### **2.1 创建分享**

*   **`POST /shares`**
*   **描述**: 创建一个新的会话分享。此接口实现了幂等性：对完全相同的对话内容重复分享，将返回之前已创建的分享链接。
*   **认证**: **必须**

*   **请求体** (`application/json`):  

| 字段 | 类型 | 必填 | 描述 |  
| :--- | :--- | :--- | :--- |  
| `messages` | `Array<Object>` | 是 | 消息对象数组，必须至少包含一条消息。 |  
| `messages[].isUser` | `Boolean` | 是 | `true` 为用户消息，`false` 为 AI 消息。 |  
| `messages[].content`| `String` | 是 | 消息内容，建议前端进行初步裁剪，后端将进行XSS清理。 |  
| `originalConversationId`| `String` | 是 | 原始对话的ID。 |

*   **请求体示例**:
    ```json
    {
      "messages": [
        { "isUser": true, "content": "你好，什么是RESTful API？" },
        { "isUser": false, "content": "RESTful API是一种..." }
      ],
      "originalConversationId": "conv_7a8b9c0d"
    }
    ```

*   **响应**:
    *   **`201 Created` - 成功创建新的分享**
    *   **`200 OK` - 找到已存在的相同内容分享**
        ```json
        {
          "data": {
            "shareId": "aB3cD_eF5gH6",
            "shareUrl": "https://yourdomain.com/share/aB3cD_eF5gH6"
          }
        }
        ```
    *   **`400 Bad Request`**: `messages` 数组为空或格式错误。
    *   **`401 Unauthorized`**: 未登录。

---

#### **2.2 获取分享数据**

*   **`GET /shares/:shareId`**
*   **描述**: 获取指定分享的详细内容，供前端预览页渲染使用。
*   **认证**: 无

*   **路径参数**:  

| 参数 | 类型 | 描述 |  
| :--- | :--- | :--- |  
| `shareId` | `String` | 分享的唯一标识符。 |  

*   **响应**:
    *   **`200 OK` - 成功**
        ```json
        {
          "data": {
            "shareId": "aB3cD_eF5gH6",
            "messages": [
              { "isUser": true, "content": "你好，什么是RESTful API？" },
              { "isUser": false, "content": "RESTful API是一种..." }
            ],
            "createdAt": "2023-10-28T10:00:00.000Z"
          }
        }
        ```
    *   **`404 Not Found`**: `shareId` 不存在或分享已被删除。

---

#### **2.3 获取我的分享列表**

*   **`GET /shares/my`**
*   **描述**: 获取当前登录用户创建的所有分享，按创建时间降序分页返回。
*   **认证**: **必须**

*   **查询参数**:  

| 参数 | 类型 | 默认值 | 描述 |  
| :--- | :--- | :--- | :--- |  
| `page` | `Integer` | 1 | 页码，从1开始。 |  
| `limit`| `Integer` | 10 | 每页数量，最大50。 |

*   **响应**:
    *   **`200 OK` - 成功**
        ```json
        {
          "data": {
            "shares": [
              {
                "shareId": "aB3cD_eF5gH6",
                "firstMessagePreview": "你好，什么是RESTful API？",
                "viewCount": 128,
                "createdAt": "2023-10-28T10:00:00.000Z",
                "expiresAt": null
              }
            ],
            "pagination": {
              "currentPage": 1,
              "totalPages": 5,
              "totalItems": 48
            }
          }
        }
        ```

---

#### **2.4 删除分享**

*   **`DELETE /shares/:shareId`**
*   **描述**: 软删除一个由当前用户创建的分享。
*   **认证**: **必须**

*   **路径参数**:  

| 参数 | 类型 | 描述 |  
| :--- | :--- | :--- |  
| `shareId` | `String` | 要删除的分享的ID。 |

*   **响应**:
    *   **`204 No Content` - 成功** (响应体为空)
    *   **`403 Forbidden`**: 尝试删除不属于自己的分享。
    *   **`404 Not Found`**: `shareId` 不存在。

---

#### **2.5 更新分享**

*   **`PATCH /shares/:shareId`**
*   **描述**: 更新一个分享的属性，例如设置或清除其有效期。
*   **认证**: **必须**

*   **路径参数**:  

| 参数 | 类型 | 描述 |  
| :--- | :--- | :--- |  
| `shareId` | `String` | 要更新的分享的ID。 |

*   **请求体** (`application/json`):

| 字段 | 类型 | 必填 | 描述 |  
| :--- | :--- | :--- | :--- |  
| `expiresAt` | `String` or `null` | 否 | ISO 8601格式的日期字符串，或 `null` 来清除有效期。 |

*   **请求体示例**:
    ```json
    {
      "expiresAt": "2025-01-01T00:00:00.000Z"
    }
    ```

*   **响应**:
    *   **`200 OK` - 成功**
        ```json
        {
          "data": {
            "shareId": "aB3cD_eF5gH6",
            "expiresAt": "2025-01-01T00:00:00.000Z"
          }
        }
        ```
    *   **`403 Forbidden`**: 尝试更新不属于自己的分享。
    *   **`404 Not Found`**: `shareId` 不存在。

### **3. 非数据接口（智能路由）**

---

#### **3.1 访问分享页**

*   **`GET /share/:shareId`**
*   **描述**: 这是用户访问分享链接的统一入口，**不是一个标准的RESTful API**。它根据访问者身份返回 HTML 页面或重定向。
*   **认证**: `Optional`

*   **查询参数**:

| 参数 | 类型 | 描述 |  
| :--- | :--- | :--- |  
| `mode` | `String` | 可选。`stream`: 指示前端使用打字机回放效果。 |

*   **行为**:
    *   如果访问者是分享的创建者，返回 **`302 Found`** 重定向到应用内对话页。
    *   如果访问者是其他人或未登录，返回 **`200 OK`** 并携带预览页的 HTML 内容。前端将通过此页面中的 JS 逻辑调用 `/api/v1/shares/:shareId` 获取数据并根据 `mode` 参数进行渲染。
    *   如果 `shareId` 无效，返回 **`404 Not Found`** 页面。
