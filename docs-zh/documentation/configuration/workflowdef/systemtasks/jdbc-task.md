---
description: "在 Conductor 中配置 JDBC 任务，针对关系型数据库执行 SQL 查询和更新。支持带连接池的 SELECT、UPDATE 和参数化查询。"
---

# JDBC 任务

```json
"type" : "JDBC"
```

JDBC 任务（`JDBC`）针对关系型数据库执行 SQL 语句。它支持 SELECT 查询、UPDATE/INSERT/DELETE 语句、参数化查询，以及失败时自动回滚的事务管理。

可以配置多个具名数据库连接，使工作流能够在同一工作流内与不同的数据库（MySQL、PostgreSQL、Oracle 等）交互。

## 任务参数

| 参数          | 类型         | 说明                                       | 必填 / 可选  |
| ------------------ | ------------ | ------------------------------------------------- | -------------------- |
| connectionId       | String       | 要使用的已配置 JDBC 实例的名称。必须与 `conductor.jdbc.instances` 配置中的某个名称匹配。 | Required（除非使用 `integrationName`）。 |
| integrationName    | String       | 托管集成（多租户）的名称。对于平台管理的连接，使用它替代 `connectionId`。 | Optional. |
| type               | String       | SQL 操作类型。支持：`SELECT`、`UPDATE`。 | Required. |
| statement          | String       | 要执行的 SQL 语句。参数化查询使用 `?`。 | Required. |
| parameters         | List[String] | 语句中 `?` 占位符的参数值有序列表。 | Optional. |
| expectedUpdateCount | Integer     | 仅适用于 `UPDATE` 类型。如果指定，当实际更新计数不匹配时，事务会被回滚。 | Optional. |
| schemaName         | String       | 数据库 schema 名称（保留，供将来使用）。 | Optional. |

## 配置 JSON

### SELECT 查询

```json
{
  "name": "query_users",
  "taskReferenceName": "query_users_ref",
  "type": "JDBC",
  "inputParameters": {
    "connectionId": "mysql-prod",
    "type": "SELECT",
    "statement": "SELECT id, name, email FROM users WHERE status = ?",
    "parameters": ["active"]
  }
}
```

### 带期望计数的 UPDATE

```json
{
  "name": "update_order_status",
  "taskReferenceName": "update_order_ref",
  "type": "JDBC",
  "inputParameters": {
    "connectionId": "mysql-prod",
    "type": "UPDATE",
    "statement": "UPDATE orders SET status = ? WHERE order_id = ?",
    "parameters": [
      "shipped",
      "${workflow.input.orderId}"
    ],
    "expectedUpdateCount": 1
  }
}
```

## 输出

### SELECT 输出

| 名称   | 类型 | 说明 |
| ------ | ---- | ----------- |
| result | List[Map[String, Any]] | 行列表，每行是列名到值的映射。 |

输出示例：

```json
{
  "result": [
    {"id": 1, "name": "Alice", "email": "alice@example.com"},
    {"id": 2, "name": "Bob", "email": "bob@example.com"}
  ]
}
```

### UPDATE 输出

| 名称   | 类型 | 说明 |
| ------ | ---- | ----------- |
| update_count | Integer | 语句影响的行数。 |

输出示例：

```json
{
  "update_count": 1
}
```

## 事务行为

- **SELECT** 语句在自动提交开启的情况下运行（JDBC 默认行为）。
- **UPDATE** 语句在自动提交关闭的情况下运行。成功时提交事务。
- 如果设置了 `expectedUpdateCount` 且实际计数不匹配，事务会被**自动回滚**，任务失败。
- 如果在 UPDATE 期间发生 SQL 异常，事务会被**自动回滚**。

## 连接配置

JDBC 连接通过 `conductor.jdbc.instances` 下的具名实例进行配置。

### 快速配置

```yaml
conductor:
  jdbc:
    instances:
      - name: "mysql-prod"
        connection:
          datasourceURL: "jdbc:mysql://prod-db:3306/myapp"
          jdbcDriver: "com.mysql.cj.jdbc.Driver"
          user: "conductor"
          password: "secret"
          maximumPoolSize: 20

      - name: "postgres-analytics"
        connection:
          datasourceURL: "jdbc:postgresql://analytics-db:5432/warehouse"
          user: "analyst"
          password: "secret"
```

### 连接池选项

| 属性 | 类型 | 默认值 | 说明 |
|----------|------|---------|-------------|
| `datasourceURL` | String | Required | JDBC 连接 URL |
| `jdbcDriver` | String | 自动检测 | JDBC 驱动类名 |
| `user` | String | Optional | 数据库用户名 |
| `password` | String | Optional | 数据库密码 |
| `maximumPoolSize` | Integer | 32 | 池中最大连接数 |
| `minimumIdle` | Integer | 2 | 最小空闲连接数 |
| `idleTimeoutMs` | Long | 30000 | 空闲连接超时（毫秒） |
| `connectionTimeout` | Long | 30000 | 连接获取超时（毫秒） |
| `leakDetectionThreshold` | Long | 60000 | 连接泄漏检测阈值（毫秒） |
| `maxLifetime` | Long | 1800000 | 连接最大生命周期（毫秒） |

## 执行

JDBC 任务按以下方式完成：

- **COMPLETED**：SQL 语句执行成功。对于 SELECT，结果在 `output.result` 中。对于 UPDATE，计数在 `output.update_count` 中。
- **FAILED**：任务在以下情况失败：
    - `connectionId` 与任何已配置实例都不匹配。
    - 发生 SQL 异常（语法错误、约束违规、连接超时）。
    - `expectedUpdateCount` 与实际更新计数不匹配（仅限 UPDATE，触发回滚）。

## 示例

### 参数化 SELECT

```json
{
  "name": "find_active_orders",
  "taskReferenceName": "find_orders_ref",
  "type": "JDBC",
  "inputParameters": {
    "connectionId": "postgres-analytics",
    "type": "SELECT",
    "statement": "SELECT order_id, total, created_at FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC",
    "parameters": [
      "${workflow.input.customerId}",
      "active"
    ]
  }
}
```

### 带期望计数的 INSERT

```json
{
  "name": "create_audit_record",
  "taskReferenceName": "audit_ref",
  "type": "JDBC",
  "inputParameters": {
    "connectionId": "mysql-prod",
    "type": "UPDATE",
    "statement": "INSERT INTO audit_log (action, user_id, details, created_at) VALUES (?, ?, ?, NOW())",
    "parameters": [
      "${workflow.input.action}",
      "${workflow.input.userId}",
      "${workflow.input.details}"
    ],
    "expectedUpdateCount": 1
  }
}
```

### 串联 SELECT 和 UPDATE

将 SELECT 任务的输出用作 UPDATE 任务的输入：

```json
[
  {
    "name": "get_order",
    "taskReferenceName": "get_order_ref",
    "type": "JDBC",
    "inputParameters": {
      "connectionId": "mysql-prod",
      "type": "SELECT",
      "statement": "SELECT id, total FROM orders WHERE order_id = ?",
      "parameters": ["${workflow.input.orderId}"]
    }
  },
  {
    "name": "apply_discount",
    "taskReferenceName": "apply_discount_ref",
    "type": "JDBC",
    "inputParameters": {
      "connectionId": "mysql-prod",
      "type": "UPDATE",
      "statement": "UPDATE orders SET total = total * 0.9 WHERE order_id = ? AND total > 0",
      "parameters": ["${workflow.input.orderId}"],
      "expectedUpdateCount": 1
    }
  }
]
```

### 在同一工作流中使用不同数据库

```json
[
  {
    "name": "read_from_mysql",
    "taskReferenceName": "mysql_read_ref",
    "type": "JDBC",
    "inputParameters": {
      "connectionId": "mysql-prod",
      "type": "SELECT",
      "statement": "SELECT user_id, email FROM users WHERE user_id = ?",
      "parameters": ["${workflow.input.userId}"]
    }
  },
  {
    "name": "write_to_postgres",
    "taskReferenceName": "pg_write_ref",
    "type": "JDBC",
    "inputParameters": {
      "connectionId": "postgres-analytics",
      "type": "UPDATE",
      "statement": "INSERT INTO user_activity (user_id, email, event_type, event_time) VALUES (?, ?, ?, NOW())",
      "parameters": [
        "${workflow.input.userId}",
        "${mysql_read_ref.output.result[0].email}",
        "workflow_triggered"
      ],
      "expectedUpdateCount": 1
    }
  }
]
```

!!! warning "SQL 注入"
    始终使用参数化查询（`?` 占位符配合 `parameters` 列表）。切勿将用户输入直接拼接到 SQL 语句中。
