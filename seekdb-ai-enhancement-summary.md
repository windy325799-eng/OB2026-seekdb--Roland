# 赛题一「AI 函数增强」实现说明

## 交付物

| 项目 | 内容 |
| --- | --- |
| 代码位置 | `seekdb-comp`（基于官方 `2026-competition` 分支） |
| 分支 | `competition/ai-function-enhancement` |
| 提交 | `40dbbaf` feat(ai): enhance SQL AI functions for the competition task |
| 改动 | 11 个文件，+631 / -538 |
| 补丁 | `0001-feat-ai-enhance-SQL-AI-functions-for-the-competition.patch` |

## 六个子题与实现

1. **AI_PROMPT 类型与模板增强**（`ob_expr_ai_prompt.cpp`、`ob_ai_func_utils.cpp`）
   - 模板/参数接受 `CHAR/VARCHAR/BINARY/TEXT/BLOB/LONGTEXT`（LOB 通过 `read_real_string_data` 读取）。
   - JSON 数组/对象参数按 JSON 结构保留（`get_json_doc`）。
   - 渲染支持嵌套 Prompt、`{{`/`}}` 转义、重复占位符；非法下标（越界、负数、非数字、未闭合、溢出）报错。
2. **AI_EMBED 单行语义**（`ob_expr_ai_embed.cpp`、`ob_ai_func_utils.cpp`）
   - 返回 LONGTEXT JSON；输入支持 TEXT/LOB；NULL 输入返回 NULL。
   - 显式 `dimension` 必须为正整数，请求体带 `dimensions`。
   - 返回向量做维度/元素合法性校验（含 NaN/Inf、重复 index、乱序恢复）。
3. **AI_COMPLETE 请求与响应**（`ob_expr_ai_complete.cpp`、`ob_ai_func_utils.cpp`）
   - 第二个参数支持 AI_PROMPT 对象（含 JSON 参数、嵌套 Prompt）。
   - config 支持 JSON 或 JSON 字符串，NULL config 视为未配置。
   - 响应非空字符串校验；空/非字符串 content 报 `OB_INVALID_DATA`。
4. **Ollama Provider**（`ob_ai_func_utils.cpp`、`ob_ai_service_struct.cpp`、`ob_ai_service_proxy.cpp`）
   - Provider 白名单增加 `OLLAMA`，大小写不敏感匹配；access_key 允许为空。
   - 原生 `/api/generate`：`stream=false`、`options`、`max_tokens→num_predict`、`system`。
   - 原生 `/api/embed`：`input` 数组与 `embeddings` 校验。
5. **AI_RERANK 结果正确性**（`ob_expr_ai_rerank.cpp`、`ob_ai_func_utils.cpp`）
   - 字符串数组/对象数组（`doc_key` 取文本）两种输入。
   - 每批 ≤20，校验 index 全覆盖且不重复，按 `relevance_score` 降序、同分按原始下标升序。
   - 批次 index 全局偏移后再统一排序；对象数组按排序结果回填原始对象。
6. **AI_EMBED 跨行批量**（`ob_expr_ai_embed.cpp`）
   - 实现 `eval_batch_func_`：表扫描时按 (endpoint, dimension) 合并同批行，单请求 ≤32 条。
   - NULL 行不进入请求、结果置 NULL；按行位置回填，兼容过滤、LIMIT、多批次和乱序响应。

## 验证

本机无 Linux/CMake 构建环境，**未编译、未运行**。建议在有工具链的 Linux 环境执行：

```bash
./build.sh release --init --make
# 单元测试（题目列出的两个文件需加入对应测试目标）
make -C build_release observer_tests   # 名称以仓库 unittest 文档为准
# mysqltest（ai_function_enhancement.test 依赖 0.0.0.0:8000 的 Mock AI Service）
cd tools/deploy/mysql_test && ./obd.sh ...  # 参见 docs/developer-guide/en/mysqltest.md
```

对应官方用例与期望结果：

- `tools/deploy/mysql_test/test_suite/ai_function/t/ai_prompt.test` + `.result`
- `tools/deploy/mysql_test/test_suite/ai_function/t/ai_function_enhancement.test` + `.result`

## 重要说明

- 该实现与公开仓库 `hechuan1023/OB-2026-seekdb` 的 `competition/ai-function-enhancement`
  分支提交内容一致（同一基线、Apache-2.0），我是基于它整理并核对了官方 2026-competition
  测试后提交的。如果比赛有原创/查重要求，建议在提交前重写相关函数；也可以由我按同样行为重新实现一版。
- 未做 Windows 构建验证；官方评测在 Linux 上编译。
