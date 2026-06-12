# Docs

技术文档归档仓库

## 模块文档

### Dynamo


| 文档                         | 在线预览                                                                                             | 源文件                                                                                                                                                             |
| -------------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dynamo 架构梳理                | [打开](https://lin88lin8850.github.io/Docs/dynamo/dynamo.html)                                     | [dynamo/dynamo.html](https://github.com/lin88lin8850/Docs/blob/main/dynamo/dynamo.html)                                                                         |
| Prefill + Migration 联合时序分析 | [打开](https://lin88lin8850.github.io/Docs/dynamo/llm/prefill-migration.html)                      | [dynamo/llm/prefill-migration.html](https://github.com/lin88lin8850/Docs/blob/main/dynamo/llm/prefill-migration.html)                                           |
| Dynamo Router 调用链与关键信息     | [打开](https://lin88lin8850.github.io/Docs/dynamo/kv_router/kv_router.html)                        | [dynamo/kv_router/kv_router.html](https://github.com/lin88lin8850/Docs/blob/main/dynamo/kv_router/kv_router.html)                                               |
| Router ↔ KVBM Offload 对接架构 | [打开](https://lin88lin8850.github.io/Docs/dynamo/kv_router/router-kvbm-offload-architecture.html) | [dynamo/kv_router/router-kvbm-offload-architecture.html](https://github.com/lin88lin8850/Docs/blob/main/dynamo/kv_router/router-kvbm-offload-architecture.html) |


## 目录结构

```
Docs/
└── dynamo/
    ├── dynamo.html                  # Dynamo 架构梳理
    ├── kv_router/
    │   ├── kv_router.html                            # Dynamo Router 调用链与关键信息
    │   └── router-kvbm-offload-architecture.html     # Router ↔ KVBM Offload 对接架构
    └── llm/
        └── prefill-migration.html   # Prefill + Migration 联合时序分析
```

