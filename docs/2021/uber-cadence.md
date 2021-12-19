---
title: Uber Cadence工作流引擎
icon: material/emoticon-happy
---

# Uber Cadence工作流引擎



最近一年（2021）一直忙于任务调度相关的工作，从最开始的业务代码逻辑实现工作流，慢慢走向开源工作流引擎的使用


## 工作流

**Temporal**的一篇博文[Designing A Workflow Engine from First Principles](https://docs.temporal.io/blog/workflow-engine-principles/#tailwind)描述了如何设计工作流系统：

### 1. 状态机


``` mermaid
graph LR
  A[Start] --> B{Error?};
  B -->|Yes| C[Hmm...];
  C --> D[Debug];
  D --> B;
  B ---->|No| E[Yay!];
```



### 2. 任务队列

### 3. 定时器

### 4. 一致性与事务

## Celery

如果

## Uber Cadence

学习[Cadence](https://cadenceworkflow.io)的最佳方式是去学习[Temporal](https://temporal.io)，不得不说cadence的模型抽象不同于常规任务队列方式实现的，
学习的过程中官方文档帮助并没有后来者`Temporal`

`cadence`的编程模型建立在两个极为重要的概念上：**Workflow**和**Activity**


### Workflow

在一些其他的工作流框架中，workflow可能是以DSL的形式存在，如Netflix的conductor提供的workflow描述语言。

```json
{
  "name": "workflow-sample",
  "description": "workflow sample",
  "version": 1,
  "tasks": [
    {
      "name": "encode",
      "taskReferenceName": "encode",
      "type": "SIMPLE",
      "inputParameters": {
        "fileLocation": "${workflow.input.fileLocation}"
      }
    },
    {
      "name": "switch_task",
      "taskReferenceName": "switch",
      "inputParameters": {
          "case_value_param": "${workflow.input.movieType}"
      },
      "type": "SWITCH",
      "evaluatorType": "value-param",
      "expression": "case_value_param",
      "decisionCases": {
        "Show": [
          {
            "name": "setup_episodes",
            "taskReferenceName": "se1",
            "inputParameters": {
              "movieId": "${workflow.input.movieId}"
            },
            "type": "SIMPLE"
          },
          {
            "name": "generate_episode_artwork",
            "taskReferenceName": "ga",
            "inputParameters": {
              "movieId": "${workflow.input.movieId}"
            },
            "type": "SIMPLE"
          }
        ],
        "Movie": [
          {
            "name": "setup_movie",
            "taskReferenceName": "sm",
            "inputParameters": {
              "movieId": "${workflow.input.movieId}"
            },
            "type": "SIMPLE"
          },
          {
            "name": "generate_movie_artwork",
            "taskReferenceName": "gma",
            "inputParameters": {
              "movieId": "${workflow.input.movieId}"
            },
            "type": "SIMPLE"
          }
        ]
      }
    }
  ],
  "outputParameters": {
    "cdn_url": "${d1.output.location}"
  },
  "failureWorkflow": "cleanup_encode_resources",
  "restartable": true,
  "workflowStatusListenerEnabled": true,
  "schemaVersion": 2, 
  "ownerEmail": "foo@bar.com"
}
```

但`cadence`并不直接提供DSL方式，而是提供编程接口。牺牲了简单易用的直接DSL方式，换来的是强大的灵活的功能。

工作流

```go
func PaymentWorkflow(ctx workflow.Context, userID string, intervals []int) error {
  // Send reminder emails, e.g. after 1, 7, and 30 days
  for _, interval := range intervals {
    _ = workflow.Sleep(ctx, days(interval)) // Sleep for days!
    _ = workflow.ExecuteActivity(ctx, SendEmail, userID).Get(ctx, nil) 
     // Activities have timeouts, and will be retried by default!
  }
  // ...
 }
```

### Activity

`cadence`