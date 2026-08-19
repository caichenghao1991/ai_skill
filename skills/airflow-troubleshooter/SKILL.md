---
name: airflow-troubleshooter
version: 3.1.0
description: Airflow/K8s/Helm 问题诊断助手，自动从 kubeconfig 和部署文件提取参数，支持规则复用、Token 优化、持久化统计及自动化脚本提醒。
author: [Your Name]
platforms: [claude, gpt, deepseek, gemini]
tags: [airflow, kubernetes, helm, troubleshooting, devops, sre, knowledge-base, persistence]
category: devops
license: MIT
created: 2026-08-19
updated: 2026-08-19

Airflow Troubleshooter — 智能诊断与知识积累 Skill
🎯 概述
本 Skill 帮助用户快速定位 Airflow 部署在 Kubernetes 环境中的问题根源。核心设计：

自动参数提取：从 kubeconfig 读取 namespace，从部署 Values 文件提取数据库连接参数（host/port/user/database），无需用户重复输入。

智能规则复用：诊断结论结构化存储，新问题优先匹配已有规则，大幅节省 Token。

持久化统计：YAML 文件记录问题出现频率，精确触发脚本生成提醒（出现≥3次且占比>30%）。

只读安全：所有诊断命令仅限于查询，修改操作须用户明确授权。

🔧 核心能力
多源诊断：读取 PostgreSQL（Airflow 元数据库）和 Kubernetes 集群状态（仅查询）。

自动参数注入：从 kubeconfig 提取 namespace，从 Values 文件提取数据库参数。

规则存储与匹配：将成功诊断的问题模式、原因、解决方案结构化存储，后续直接匹配引用。

持久化统计：YAML 文件记录每次诊断的使用次数和总调查数。

一次性信息收集：在正式执行命令前，一次性询问所有必要输入（文件路径、症状），避免多轮对话。

自动化脚本提醒：当某类问题满足阈值时，提示用户编写自动化脚本，并提供代码骨架。

📥 输入规范（用户只需提供 4 项）
用户启动本 Skill 时，需提供以下信息（一次性提供）：

输入项	说明	示例
数据库连接文件	YAML 格式，含 host, port, user, password, database；或 Helm Values 文件（含 postgresql 配置）	/path/to/db.yaml 或 /path/to/values.yaml
K8s 凭证文件	kubeconfig 格式，用于连接集群	~/.kube/config
Helm Chart 文件夹	本地目录路径，用于检查 Helm 配置	/path/to/airflow-helm-chart
问题症状描述	自然语言描述具体表现	“DAG daily_report 在 Web UI 中不显示”
自动提取：

namespace：从 kubeconfig 的 current-context 中读取（若未设置则用 default）。

数据库参数：从 Values 文件（若提供）中解析（按约定字段如 postgresql.host、postgresql.port 等），或直接从数据库 YAML 读取。

📤 输出规范
格式：Markdown 报告

长度：800 字以内

结构：

问题症状复述
关键诊断发现（附证据摘要）
根本原因
解决方案（修改操作标注“需确认”）
规则匹配/存储状态（若引用规则，列出规则 ID）
脚本提醒（若满足阈值）
📖 执行流程（指令）
阶段 0：一次性信息收集（强制）
AI 必须一次性询问以下 4 项信息（合并为一个问题列表）：

text
请一次性提供以下信息以开始诊断：
1. 数据库连接文件路径（YAML 格式）或 Helm Values 文件路径（含 postgresql 配置）
2. K8s 凭证文件路径（kubeconfig）
3. Helm Chart 文件夹路径（本地目录）
4. 问题症状描述（自然语言）
禁止分多次提问。

阶段 1：解析与初始化（自动提取参数）
读取 kubeconfig：

解析 current-context，提取 namespace（若未设置，默认 default）。

验证凭证有效性（kubectl cluster-info）。

读取数据库参数：

若提供的是 Values YAML 文件，按以下顺序查找数据库配置（根据实际结构灵活匹配）：

postgresql.host / postgresql.port / postgresql.postgresqlUsername / postgresql.postgresqlPassword / postgresql.postgresqlDatabase

或 airflow.database.host / airflow.database.port / airflow.database.user / airflow.database.password / airflow.database.database

若提供的是纯数据库 YAML，直接读取 host、port、user、password、database。

切换 namespace：

text
kubectl config set-context --current --namespace=<提取的 namespace>
后续 kubectl 和 helm 命令均无需再加 -n 参数。

检查持久化存储路径：

若首次使用，询问用户存储路径（用于存放 troubleshoot_stats.yaml）；后续沿用。

若路径不存在则创建（需具备读写权限）。

读取 YAML 统计文件（若存在），获取历史数据。

阶段 2：规则匹配（Token 优化）
查询长期记忆（memory_save）中所有 reference 类型且标签含 airflow-troubleshoot-rule 的规则。

提取当前问题的症状关键词，与已有规则的 symptom_patterns 进行模糊匹配（基于关键词重合度）。

若匹配度 ≥ 80%：

输出该规则对应的解决方案，跳过阶段 3-4（直接进入阶段 5）。

在报告中注明“已引用规则 ID: xxx”。

更新统计：该规则对应的 count 加 1，total_investigations 加 1。

阶段 3：执行诊断命令（仅读取）
按顺序执行以下命令（因已切换 namespace，kubectl 和 helm 命令不加 -n）：

A. K8s 资源状态检查
text
kubectl get pods
kubectl get svc
kubectl get deployment
kubectl get configmap
kubectl describe pod <可疑 Pod 名>
kubectl logs <Pod 名> --tail=200
B. Airflow 组件检查（根据部署方式）
text
kubectl get pods -l component=scheduler
kubectl get pods -l component=webserver
kubectl get pods -l component=worker
kubectl logs <scheduler-pod> --tail=500 | grep -i error
C. 数据库查询（PostgreSQL）
text
psql -h <host> -p <port> -U <user> -d <database> -c "SELECT * FROM dag;"
psql -h <host> -p <port> -U <user> -d <database> -c "SELECT * FROM dag_run ORDER BY start_date DESC LIMIT 20;"
psql -h <host> -p <port> -U <user> -d <database> -c "SELECT * FROM task_instance WHERE state='failed' OR state='up_for_retry' LIMIT 30;"
D. Helm 配置检查
text
helm list
helm get values <release-name>
E. DAG 文件检查（如用户提供）
若用户提供 DAG 文件路径，检查文件语法。

检查 schedule_interval、depends_on_past、max_active_runs 等配置。

注意：所有命令输出应只保留关键信息（错误行、异常状态），避免输出完整长日志造成 Token 浪费。

阶段 4：分析与推理
基于收集到的证据，结合已有规则（若部分相关）进行推理。参考常见模式（但不局限于此）：

症状	常见原因
DAG 在 Web UI 不显示	DAG 文件语法错误、scheduler 未扫描到、DAG 未启用、变量依赖缺失
任务失败后重试成功	资源临时不足（内存/CPU）、网络抖动、外部依赖超时
任务一直处于 running	Worker 资源耗尽、任务死锁、数据库连接池满
调度器无响应	数据库连接问题、调度器进程挂起、Zombie 任务堆积
Helm 升级失败	Values 语法错误、K8s 资源冲突、存储类问题
AI 应根据实际数据推理，不局限于上述列表。

阶段 5：输出报告与知识存储
生成 800 字以内的 Markdown 报告，包含：

问题症状复述
关键诊断发现（附证据摘要）
根本原因
解决方案（修改操作标注“需确认”）
规则匹配/存储状态（若引用规则，列出规则 ID）
若用户确认诊断正确：

生成新规则，调用 memory_save 存储，格式如下：

type: reference
name: "airflow-rule-<short-description>"
content: |
  symptom_patterns: [关键词列表]
  root_cause: 根本原因描述
  solution_steps: [步骤列表]
  related_components: [scheduler, webserver, worker, db, k8s]
tags: ["airflow-troubleshoot-rule", "k8s", "postgres"]
将规则摘要附加在报告末尾。
更新持久化统计 YAML 文件：
增加该症状的 count（若新症状则新建条目）。
total_investigations 加 1。
保存 YAML 文件。
阶段 6：脚本生成提醒（基于统计）
从 YAML 文件中读取当前症状的 count 和 total_investigations。

计算比例：ratio = count / total_investigations。

若 count ≥ 3 且 ratio > 30%：

在报告中附加建议：💡 该问题已出现多次（≥3 次，占比 >30%），建议编写自动化诊断脚本。

提供约 150~200 行的 Python 脚本骨架（包含 argparse、kubectl/psql 调用、规则匹配逻辑），用户可直接填充具体检查项。

⚠️ 安全与隐私约束
只读原则：所有 kubectl 和 psql 操作仅限于查询（get、describe、logs、SELECT）。

修改确认：任何涉及更改的操作（如 kubectl delete、UPDATE SQL）必须先提出并等待用户明确授权，不可自动执行。

敏感信息保护：数据库密码、token 等敏感信息不记录在报告或记忆体（如需记录，仅保存脱敏版本）。

🔒 限制说明
本 Skill 需要 AI 环境具备 kubectl 和 psql 命令行工具且与目标集群/数据库连通。

若用户未提供 DAG 文件，AI 仅能从数据库和 K8s 层推理，DAG 逻辑问题可能无法完全定位。

诊断结果仅供参考，生产环境变更需由用户评估后执行。

持久化统计和规则存储依赖于用户提供的本地文件夹路径，且该文件夹需具备读写权限。

💡 使用示例
首次使用（一次性提问）
AI 输出：

text
请一次性提供以下信息以开始诊断：
1. 数据库连接文件路径（YAML 格式）或 Helm Values 文件路径（含 postgresql 配置）
2. K8s 凭证文件路径（kubeconfig）
3. Helm Chart 文件夹路径（本地目录）
4. 问题症状描述（自然语言）
用户回复（一次性）：

text
1. /home/user/airflow-values.yaml
2. /home/user/.kube/config
3. /home/user/airflow-helm
4. DAG daily_report 在 Web UI 中不显示
AI 自动提取：

从 config 读取 namespace: airflow-prod

从 values.yaml 读取数据库参数: host=postgres-airflow, port=5432, user=airflow, database=airflow_metadata

AI 响应（诊断后）：

text
## 问题症状
DAG daily_report 在 Web UI 中不显示。

## 诊断发现
1. 数据库查询显示 `is_paused` 为 `True`。
2. scheduler 日志报错：`File /dags/daily_report.py has no DAGs`。
3. DAG 文件中 `dag_id` 与函数名不一致。

## 根本原因
dag_id 与函数名不匹配，scheduler 无法解析。

## 解决方案
1. 修改 DAG 文件，使 `dag_id` 与函数名一致。
2. 重新上传并等待扫描。
---
✅ 已保存规则（ID: 1001），后续同类问题将自动引用。
✅ 质量检查清单
□ 一次性提问是否完整（4 项）？
□ 是否从 kubeconfig 自动提取 namespace？
□ 是否从 Values 文件自动提取数据库参数？
□ 规则匹配阈值是否设为 ≥80%？
□ 脚本触发条件是否正确（count≥3 且 ratio>30%）？
□ 所有输出是否控制在 800 字以内？
📎 附录 A：持久化 YAML 格式模板
text
# 存储于用户指定文件夹下的 troubleshoot_stats.yaml
total_investigations: 0
symptom_patterns:
  dag_not_display:
    count: 0
  task_fail_retry:
    count: 0
📎 附录 B：数据库配置提取映射表（用于 Values 文件）
Values 字段路径	对应参数
postgresql.host	host
postgresql.port	port
postgresql.postgresqlUsername	user
postgresql.postgresqlPassword	password
postgresql.postgresqlDatabase	database
airflow.database.host	host
airflow.database.port	port
airflow.database.user	user
airflow.database.password	password
airflow.database.database	database
结束语：此 Skill 自动提取参数、智能复用规则，大幅减少重复输入和 Token 消耗。保存为 airflow-troubleshooter.txt 即可使用。