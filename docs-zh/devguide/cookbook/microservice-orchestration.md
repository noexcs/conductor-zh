---
description: "Conductor 手册（cookbook）— 微服务编排示例集：HTTP 服务链、条件分支，以及使用 Fork/Join 的并行 HTTP 调用。"
---

# 微服务编排

### HTTP 服务链

一个常见模式：调用一系列 HTTP 端点，每一步都使用上一步的输出。无需自定义 worker——Conductor 用内置的 HTTP 任务即可处理。

```json
{
  "name": "order_processing",
  "description": "Validate order, charge payment, reserve inventory, send confirmation",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["orderId", "customerId", "amount", "items"],
  "tasks": [
    {
      "name": "validate_order",
      "taskReferenceName": "validate",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/orders/${workflow.input.orderId}/validate",
          "method": "POST",
          "body": {
            "customerId": "${workflow.input.customerId}",
            "items": "${workflow.input.items}"
          },
          "connectionTimeOut": 5000,
          "readTimeOut": 5000
        }
      }
    },
    {
      "name": "charge_payment",
      "taskReferenceName": "payment",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/payments/charge",
          "method": "POST",
          "body": {
            "orderId": "${workflow.input.orderId}",
            "amount": "${workflow.input.amount}",
            "customerId": "${workflow.input.customerId}"
          },
          "connectionTimeOut": 10000,
          "readTimeOut": 10000
        }
      }
    },
    {
      "name": "reserve_inventory",
      "taskReferenceName": "inventory",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/inventory/reserve",
          "method": "POST",
          "body": {
            "orderId": "${workflow.input.orderId}",
            "items": "${workflow.input.items}",
            "paymentId": "${payment.output.response.body.paymentId}"
          },
          "connectionTimeOut": 5000,
          "readTimeOut": 5000
        }
      }
    },
    {
      "name": "send_confirmation",
      "taskReferenceName": "notify",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/notifications/send",
          "method": "POST",
          "body": {
            "customerId": "${workflow.input.customerId}",
            "orderId": "${workflow.input.orderId}",
            "paymentId": "${payment.output.response.body.paymentId}",
            "reservationId": "${inventory.output.response.body.reservationId}"
          }
        }
      }
    }
  ],
  "outputParameters": {
    "paymentId": "${payment.output.response.body.paymentId}",
    "reservationId": "${inventory.output.response.body.reservationId}"
  },
  "failureWorkflow": "order_compensation",
  "timeoutPolicy": "TIME_OUT_WF",
  "timeoutSeconds": 120
}
```

每个任务使用 `${taskReferenceName.output.response.body.field}` 表达式向前传递数据。如果任何一步失败，Conductor 会重试它（可配置），并可以触发 `failureWorkflow` 进行补偿。

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @order_processing.json

curl -X POST 'http://localhost:8080/api/workflow/order_processing' \
  -H 'Content-Type: application/json' \
  -d '{"orderId": "ORD-123", "customerId": "CUST-456", "amount": 99.99, "items": ["SKU-A", "SKU-B"]}'
```

---

### 带条件分支的 HTTP

使用 SWITCH 算子，根据前一个任务的输出对工作流执行进行路由。

```json
{
  "name": "user_onboarding",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["userId"],
  "tasks": [
    {
      "name": "get_user_profile",
      "taskReferenceName": "profile",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/users/${workflow.input.userId}",
          "method": "GET"
        }
      }
    },
    {
      "name": "route_by_tier",
      "taskReferenceName": "tier_switch",
      "type": "SWITCH",
      "evaluatorType": "javascript",
      "expression": "$.tier == 'enterprise' ? 'enterprise' : 'standard'",
      "inputParameters": {
        "tier": "${profile.output.response.body.tier}"
      },
      "decisionCases": {
        "enterprise": [
          {
            "name": "assign_account_manager",
            "taskReferenceName": "assign_am",
            "type": "HTTP",
            "inputParameters": {
              "http_request": {
                "uri": "https://api.example.com/account-managers/assign",
                "method": "POST",
                "body": {"userId": "${workflow.input.userId}"}
              }
            }
          }
        ],
        "standard": [
          {
            "name": "send_welcome_email",
            "taskReferenceName": "welcome",
            "type": "HTTP",
            "inputParameters": {
              "http_request": {
                "uri": "https://api.example.com/emails/welcome",
                "method": "POST",
                "body": {"userId": "${workflow.input.userId}"}
              }
            }
          }
        ]
      }
    }
  ]
}
```

---

### 使用 Fork/Join 的并行 HTTP 调用

当任务相互独立时，使用静态 fork 让它们并行运行。

```json
{
  "name": "enrich_customer_data",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["customerId"],
  "tasks": [
    {
      "name": "parallel_enrichment",
      "taskReferenceName": "fork",
      "type": "FORK_JOIN",
      "forkTasks": [
        [
          {
            "name": "get_credit_score",
            "taskReferenceName": "credit",
            "type": "HTTP",
            "inputParameters": {
              "http_request": {
                "uri": "https://api.example.com/credit/${workflow.input.customerId}",
                "method": "GET"
              }
            }
          }
        ],
        [
          {
            "name": "get_purchase_history",
            "taskReferenceName": "purchases",
            "type": "HTTP",
            "inputParameters": {
              "http_request": {
                "uri": "https://api.example.com/purchases/${workflow.input.customerId}",
                "method": "GET"
              }
            }
          }
        ],
        [
          {
            "name": "get_support_tickets",
            "taskReferenceName": "tickets",
            "type": "HTTP",
            "inputParameters": {
              "http_request": {
                "uri": "https://api.example.com/support/${workflow.input.customerId}",
                "method": "GET"
              }
            }
          }
        ]
      ]
    },
    {
      "name": "join_results",
      "taskReferenceName": "join",
      "type": "JOIN",
      "joinOn": ["credit", "purchases", "tickets"]
    }
  ],
  "outputParameters": {
    "creditScore": "${credit.output.response.body}",
    "purchases": "${purchases.output.response.body}",
    "tickets": "${tickets.output.response.body}"
  }
}
```

三个 HTTP 调用同时执行。JOIN 会等待它们全部完成后，工作流才继续。
