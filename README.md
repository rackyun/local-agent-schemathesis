# local-agent-schemathesis
本地 Agent + Schemathesis 工作流设计 这是一套完全自托管、零许可成本的方案，适合替代 Keploy / Karate Agent。核心思路是：  Schemathesis 负责从 OpenAPI 自动生成海量属性测试（正例、边界、负例、状态机） 本地 Agent（Claude Code / Cursor / Continue + 本地模型）负责补全业务规则、分析失败、生成自定义检查、维护 Schema
