# 技术设计方案

## 一、系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Flutter Web 客户端                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    presentation/                       │    │
│  │   pages/    widgets/    providers/    controllers/    │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    domain/                              │    │
│  │   entities/    repositories/    usecases/              │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    data/                                 │    │
│  │   repositories/    datasources/    models/             │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────┘
                                │ HTTP (JSON)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Go 后端服务                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    handlers/                             │    │
│  │   requirement_handler.go    ai_handler.go              │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    services/                             │    │
│  │   requirement_service.go    ai_service.go              │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    repositories/                         │    │
│  │   requirement_repo.go    ai_config_repo.go             │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    models/                               │    │
│  │   requirement.go    ai_config.go                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    middleware/                           │    │
│  │   cors.go    logger.go                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    config/                               │    │
│  │   config.go    database.go                              │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
                         ┌─────────────┐
                         │   MySQL     │
                         └─────────────┘
```

---

## 二、Go 后端目录结构

```
backend/
├── cmd/
│   └── server/
│       └── main.go              # 入口文件
├── internal/
│   ├── config/
│   │   ├── config.go            # 配置加载
│   │   └── database.go         # 数据库连接
│   ├── models/
│   │   ├── requirement.go       # 需求模型
│   │   ├── ai_config.go         # AI 配置模型
│   │   └── common.go            # 通用模型（响应结构）
│   ├── repositories/
│   │   ├── requirement_repo.go  # 需求数据访问
│   │   └── ai_config_repo.go    # AI 配置数据访问
│   ├── services/
│   │   ├── requirement_service.go  # 需求业务逻辑
│   │   └── ai_service.go        # AI 服务（调用 LLM）
│   ├── handlers/
│   │   ├── requirement_handler.go  # 需求 API 处理
│   │   └── ai_handler.go       # AI 配置 API 处理
│   └── middleware/
│       ├── cors.go              # CORS 中间件
│       └── logger.go            # 日志中间件
├── pkg/
│   └── utils/
│       └── uuid.go              # UUID 工具
├── go.mod
├── go.sum
└── .env                        # 环境变量（数据库、端口等）
```

### Go 后端核心文件说明

| 文件 | 职责 |
|------|------|
| `main.go` | 启动服务、注册路由 |
| `config.go` | 加载环境变量（MySQL、端口） |
| `requirement.go` | 需求数据结构体 |
| `requirement_handler.go` | 处理需求的 HTTP 请求 |
| `requirement_service.go` | 需求增删改查业务逻辑 |
| `requirement_repo.go` | 操作 MySQL |
| `ai_service.go` | 调用 AI 模型 API |
| `ai_handler.go` | AI 配置管理 |

### API 接口设计

| 方法 | 路径 | 说明 |
|------|------|------|
| **需求相关** |||
| GET | /api/requirements | 获取需求列表 |
| GET | /api/requirements/:id | 获取需求详情 |
| POST | /api/requirements | 创建需求 |
| PUT | /api/requirements/:id | 更新需求 |
| DELETE | /api/requirements/:id | 删除需求 |
| POST | /api/requirements/:id/ai-check | AI 检查需求 |
| **AI 配置相关** |||
| GET | /api/ai-configs | 获取 AI 配置列表 |
| POST | /api/ai-configs | 创建 AI 配置 |
| PUT | /api/ai-configs/:id | 更新 AI 配置 |
| DELETE | /api/ai-configs/:id | 删除 AI 配置 |
| GET | /api/ai-configs/default | 获取默认 AI 配置 |

---

## 三、Flutter 前端目录结构

```
frontend/
├── lib/
│   ├── main.dart                # 入口
│   ├── app.dart                 # App 根组件
│   ├── core/                   # 核心工具
│   │   ├── constants/
│   │   │   └── api.dart        # API 地址配置
│   │   ├── theme/
│   │   │   └── app_theme.dart  # 主题配置
│   │   └── utils/
│   │       └── extensions.dart # 扩展方法
│   ├── domain/                 # 领域层（业务实体）
│   │   ├── entities/
│   │   │   ├── requirement.dart   # 需求实体
│   │   │   ├── ai_config.dart     # AI 配置实体
│   │   │   └── ai_check_result.dart # AI 检查结果
│   │   └── repositories/       # 仓储接口
│   │       ├── requirement_repository.dart
│   │       └── ai_config_repository.dart
│   ├── data/                   # 数据层
│   │   ├── models/             # 数据模型（JSON 序列化）
│   │   │   ├── requirement_model.dart
│   │   │   ├── ai_config_model.dart
│   │   │   └── ai_check_result_model.dart
│   │   ├── datasources/
│   │   │   └── remote/
│   │   │       ├── requirement_datasource.dart
│   │   │       └── ai_config_datasource.dart
│   │   └── repositories/       # 仓储实现
│   │       ├── requirement_repository_impl.dart
│   │       └── ai_config_repository_impl.dart
│   ├── presentation/            # 表现层（UI）
│   │   ├── providers/           # Riverpod Provider
│   │   │   ├── requirement_provider.dart
│   │   │   ├── ai_config_provider.dart
│   │   │   └── ai_check_provider.dart
│   │   ├── pages/
│   │   │   ├── home/
│   │   │   │   └── home_page.dart       # 需求列表页
│   │   │   ├── requirement/
│   │   │   │   ├── requirement_list_page.dart
│   │   │   │   ├── requirement_edit_page.dart  # 录入/编辑
│   │   │   │   └── requirement_detail_page.dart # 详情
│   │   │   └── ai_config/
│   │   │       └── ai_config_page.dart  # AI 配置管理
│   │   └── widgets/
│   │       ├── requirement_card.dart    # 需求卡片
│   │       ├── requirement_form.dart     # 需求表单
│   │       ├── markdown_editor.dart      # Markdown 编辑器
│   │       ├── ai_check_panel.dart       # AI 检查结果面板
│   │       └── status_tag.dart           # 状态标签
│   └── router/
│       └── app_router.dart      # 路由配置
├── pubspec.yaml
└── .env                        # 环境变量（API 地址）
```

### Flutter 前端核心文件说明

| 文件 | 职责 |
|------|------|
| `requirement.dart` | 需求实体（业务模型） |
| `requirement_provider.dart` | Riverpod 状态管理 |
| `requirement_edit_page.dart` | 需求录入/编辑页面 |
| `ai_check_panel.dart` | AI 检查结果展示 |
| `markdown_editor.dart` | Markdown 编辑组件 |

---

## 四、数据模型对应

### 4.1 需求 (Requirement)

```go
// Go - models/requirement.go
type Requirement struct {
    ID            string    `json:"id" gorm:"primaryKey"`
    Title         string    `json:"title" gorm:"size:255;not null"`
    Description   string    `json:"description" gorm:"type:text"`
    DesignLink    string    `json:"design_link" gorm:"size:500"`
    Priority      string    `json:"priority" gorm:"size:10"`      // P0/P1/P2/P3
    ReqType       string    `json:"req_type" gorm:"size:20"`     // 新功能/缺陷修复/体验优化
    Status        string    `json:"status" gorm:"size:20"`        // 草稿/待检查/确认完善/待开发/开发中/已完成
    AICheckResult string    `json:"ai_check_result" gorm:"type:text"` // JSON 字符串
    CreatedBy     string    `json:"created_by" gorm:"size:50"`
    CreatedAt     time.Time `json:"created_at"`
    UpdatedAt     time.Time `json:"updated_at"`
}
```

```dart
// Flutter - domain/entities/requirement.dart
class Requirement {
    final String id;
    final String title;
    final String description;
    final String? designLink;
    final String priority;      // P0/P1/P2/P3
    final String reqType;       // 新功能/缺陷修复/体验优化
    final String status;        // 草稿/待检查/确认完善/待开发/开发中/已完成
    final AICheckResult? aiCheckResult;
    final String createdBy;
    final DateTime createdAt;
    final DateTime updatedAt;
}
```

### 4.2 AI 配置 (AIConfig)

```go
// Go - models/ai_config.go
type AIConfig struct {
    ID          string    `json:"id" gorm:"primaryKey"`
    Name        string    `json:"name" gorm:"size:100;not null"`
    Provider    string    `json:"provider" gorm:"size:50"`        // openai/claude/custom
    BaseURL     string    `json:"base_url" gorm:"size:255"`
    Model       string    `json:"model" gorm:"size:100"`
    APIKey      string    `json:"api_key" gorm:"size:255"`
    IsDefault   bool      `json:"is_default" gorm:"default:false"`
    CreatedAt   time.Time `json:"created_at"`
}
```

### 4.3 AI 检查结果 (AICheckResult)

```dart
// Flutter - domain/entities/ai_check_result.dart
class AICheckResult {
    final bool passed;
    final List<AISuggestion> suggestions;

    AICheckResult({
        required this.passed,
        required this.suggestions,
    });
}

class AISuggestion {
    final String type;      // completeness/clarity/logic/clarity/tasks
    final String message;  // 建议内容
}
```

---

## 五、关键业务流程

### 5.1 创建需求流程

```
用户填写表单
    ↓
Flutter 收集数据 (title, description, priority, req_type)
    ↓
POST /api/requirements
    ↓
Go Handler 接收请求
    ↓
Go Service 校验数据
    ↓
Go Repository 写入 MySQL
    ↓
返回创建的 Requirement
    ↓
Flutter 更新列表
```

### 5.2 AI 检查流程

```
用户在详情页点击"AI 检查"
    ↓
Flutter POST /api/requirements/:id/ai-check
    ↓
Go Handler 接收请求
    ↓
Go Service 获取默认 AI 配置
    ↓
Go Service 调用 AI 模型 API (OpenAI/Claude/自定义)
    ↓
AI 返回检查结果
    ↓
Go Service 更新 Requirement 的 ai_check_result
    ↓
返回检查结果
    ↓
Flutter 展示 AI 检查面板
```

---

## 六、环境变量

### Go 后端 (.env)

```bash
# 服务配置
SERVER_PORT=8080

# 数据库配置
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=requirements_db
```

### Flutter 前端 (.env)

```dart
// lib/core/constants/api.dart
class ApiConstants {
  static const String baseUrl = 'http://192.168.1.100:8080'; // 你的本地 IP
}
```

---

## 七、下一步

1. **创建 Go 后端项目** - 初始化项目、配置数据库
2. **创建 Flutter 前端项目** - 初始化项目、配置依赖
3. **实现核心功能** - 需求 CRUD + AI 检查

你想先从哪个开始？
