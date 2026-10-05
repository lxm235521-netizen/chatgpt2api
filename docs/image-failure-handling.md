# 图片失败处理

状态：当前

## 单一分类来源

`services/image_failure.py` 中的 `ImageFailure` 是图片执行期间的统一分类对象。它携带失败码、作用域、能力、是否可重试、HTTP 状态、错误类型、原始诊断和对外文案。执行、账号处理和 API 响应使用当前分类结果；调用日志、实时监控和历史尝试则消费其持久化的结构化失败字段，并为旧记录做兼容投影。任何一层都不应再按错误文本重新分类。

## 文本结果与失败

HTTP 400 的图片结果被分类为文本结果：`content_policy_violation`、`invalid_image_input`、`upstream_text_reply` 和 `unsupported_model`。这类结果会保留可展示的上游文本，默认不切换账号、不计账号失败。

例外是两段明确把责任指向账号的图片工具文本：命中「图片生成工具触发了生成频率限制」归为 `image_tool_rate_limited`，命中「由于我这边发生了错误，我未能生成图片」归为 `image_tool_upstream_error`。两者对外仍是 400 文本结果（`outcome` 不变、上游文本照原样返回），但通过 `FailurePolicy.switch_account` 与 `FailurePolicy.cooldown` 显式声明：当前请求可以立刻切换其他账号重试，触发账号同时进入图片冷却。识别由 `classify_image_tool_text` 统一完成，覆盖终态 assistant 文本与上游真实 HTTP 400 响应体，并按空白归一化后匹配，上游插入换行不影响结果；结构化 400 错误码优先于文本匹配。

其他 `ImageFailure` 的 `outcome` 是失败，允许执行账号切换与账号验证流程。是否实际切换还取决于账号池、尝试上限和当前设置；分类对象只给出一致的切换资格，不保证一定能找到下一个账号。

## 账号冷却

冷却是否发生由 `ImageFailure.cooldown` 声明，时长由设置 `image_account_cooldown_minutes` 决定（默认 6 分钟，最小 0，填 0 表示不冷却），由 `AccountService.mark_image_result` 这条图片结果回写的权威入口写入账号的 `image_cooldown_until`；重复触发只延长、不缩短已有截止时间。

冷却只作用于图片候选池：`AccountService._is_image_account_cooling_down` 在 `_list_ready_candidate_tokens` 与 `get_available_access_token` 的预检后复查处过滤。它刻意不改动 `_is_image_account_available`，因此冷却中的账号保持控制台的可用投影，也不会从图片模型目录里消失，文本等其他能力的取号完全不受影响。冷却到期由时间戳自动失效，无需后台清理。控制台账号行通过 `image_cooldown_at`、`image_cooldown_active` 和 `image_cooldown_reason` 展示冷却状态与恢复时刻。

常见类别包括：

| 类别 | 典型含义 | 对外状态 |
| --- | --- | --- |
| `auth_invalid` | 上游鉴权无效 | 401 |
| `upstream_rate_limited` / `image_quota_exhausted` | 限流或图片额度耗尽 | 429 |
| `image_poll_timeout` | 等待图片结果超时 | 502 |
| `image_stream_timeout` / `image_stream_interrupted` | SSE 超时或中断 | 502 |
| `image_tool_error` | 上游图片工具终态异常 | 502 |
| `image_download_failed` | 已生成但交付下载失败 | 502 |
| `no_available_account` | 当前账号池无法选择账号 | 503 |

## 诊断字段

日志和尝试详情区分三种信息：

- **对外错误**：返回给 API 调用方的安全文案。
- **上游错误**：上游结构化错误或异常摘要。
- **上游文本**：上游 assistant 返回的原始可读文本。

结构化 JSON 不会被误当作用户可读文本直接展示。终态 assistant 普通文本会保留为文本结果；当终态 JSON / 结构化字段明确表示图片工具失败时，后端归为 `image_tool_error` 或更具体的失败码。

## 图片任务与对外投影

`ImageTaskService` 持久化的任务状态是 `queued`、`running`、`success` 或 `error`。`/api/image-tasks` 的视图投影会根据原始状态、请求数量、成功数量和失败分类对外呈现 `success`、`partial_success`、`failed` 或 `text_review`。`partial_success` 用于多张请求中已有结果但未全部完成；`text_review` 是文本结果，不是前端把 400 临时改名后的失败。

更改图片失败策略时，先调整 `ImageFailure` 的策略和契约测试，再检查 API、账号切换、持久化诊断、日志、监控和 Studio 投影是否一致。
