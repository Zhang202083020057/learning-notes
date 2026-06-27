# Harness Engineering 架构与实现分析

> 研究日期：2026-06-27  
> 研究方向：Harness 软件交付平台架构设计与核心实现

---

## 一、Harness 是什么？

**Harness** 是一家成立于2016年的 DevOps/软件交付平台公司，总部位于美国旧金山。其核心定位是 **AI 原生的软件交付平台（AI Software Delivery Platform）**，将构建、测试、部署、验证、监控、优化等整个软件交付生命周期统一在一个智能平台中。

- **估值**：截至2025年12月，完成高盛领投的2.4亿美元E轮融资，估值达 **55亿美元**
- **ARR（年经常性收入）**：突破2.5亿美元
- **客户规模**：服务1000+企业客户
- **运行数据**：支撑**1.28亿次部署、8100万次构建**，保护1.2万亿次API调用

> 来源：[CSDN Harness全景对比](https://blog.csdn.net/RickyIT/article/details/161359414)

---

## 二、核心产品线

Harness 提供七大核心模块，形成完整的 DevOps 闭环：

| 模块 | 英文全称 | 核心能力 |
|---|---|---|
| **CI** | Continuous Integration | 容器化构建、Test Intelligence、智能缓存 |
| **CD** | Continuous Delivery | 多环境部署、金丝雀/蓝绿/滚动策略、OPA策略引擎 |
| **FF/FME** | Feature Flags & Management | 部署与发布解耦、渐进式发布、A/B测试 |
| **CCM** | Cloud Cost Management | 多云成本可视化、AutoStopping、异常检测 |
| **SRM** | Service Reliability Management | SLO/SLI管理、错误预算监控、自动回滚 |
| **STO** | Security Testing Orchestration | 50+安全扫描器集成、SAST/DAST/SCA |
| **IDP** | Internal Developer Portal | 基于Backstage、软件目录、自助工作流 |

---

## 三、架构设计

### 3.1 整体架构：微服务 + 四层结构

Harness 采用现代化的**微服务架构**，各组件通过**gRPC和REST API**进行通信：

**基础设施层**
- Kubernetes集群管理
- Docker运行时集成
- 存储系统：PostgreSQL（元数据）、MongoDB（非结构化数据）、Redis（缓存/会话）
- 消息队列：NATS、RabbitMQ（事件驱动通信）

**平台核心层**
- 身份认证与授权服务（OAuth2、SAML、LDAP）
- 项目管理与组织管理
- 审计日志与指标收集
- 通知服务（Email、Slack、Webhook）

**业务服务层**
- CI引擎服务（构建、测试、打包）
- CD引擎服务（部署、验证、回滚）
- 功能标记服务（A/B测试、渐进发布）
- 成本管理服务、安全测试服务、混沌工程服务

**接口层**
- Web UI（React + TypeScript）
- CLI工具（Go语言开发）
- REST API（OpenAPI规范）
- SDK库（Python、Java、Go等）

> 来源：[CSDN Harness平台深度解析](https://blog.csdn.net/m0_50709695/article/details/159773586)

### 3.2 Delegate 代理机制（最核心的架构创新）

Harness **Delegate** 是一个运行在客户基础设施中的轻量级容器或进程，是 Harness 架构中最重要的组件。

**工作原理**：
- Delegate 部署在客户VPC/私有网络内
- 通过**出站HTTPS/WebSocket**与Harness Manager通信
- **无需开放防火墙入站端口**，安全性极高
- Delegate 负责执行所有CI/CD操作

**核心特性**：
- 支持Kubernetes、Docker、Shell等多种安装方式
- 可横向扩展，自动负载均衡
- 内置缓存（Maven、npm、Docker层）加速构建
- 支持多环境隔离

**架构示意**：

```
用户环境（客户VPC）
  Harness Delegate
    |-- CI Runner（执行构建）
    |-- CD Runner（执行部署）
    |-- Artifactory 适配器
    |-- K8s API 适配器
        |
        |（出站HTTPS/WebSocket，轮询任务）
        v
Harness SaaS / Self-Managed Platform
  Harness Manager
    |-- Web UI
    |-- Pipeline Engine
    |-- Policy Engine (OPA)
```

> 来源：[CSDN Harness CI/CD+GitOps实战](https://blog.csdn.net/RickyIT/article/details/161359435)

### 3.3 Pipeline 编排引擎

Harness Pipeline引擎是其核心组件，负责解析、验证和执行流水线定义。

**关键数据结构（Go语言）**：

```go
// Pipeline 结构定义流水线的核心元数据
type Pipeline struct {
    ID        string                 `json:"id"`
    Name      string                 `json:"name"`
    ProjectID string                 `json:"project_id"`
    OrgID     string                 `json:"org_id"`
    Version   int64                  `json:"version"`
    YAML      string                 `json:"yaml"`
    Stages    []Stage                `json:"stages"`
    Variables map[string]interface{} `json:"variables"`
}

// Stage 表示流水线中的一个阶段
type Stage struct {
    ID        string     `json:"id"`
    Name      string     `json:"name"`
    Type      StageType  `json:"type"`
    Parallel  bool       `json:"parallel"`
    DependsOn []string   `json:"depends_on"`
    Steps     []Step     `json:"steps"`
    Timeout   time.Duration `json:"timeout"`
}

// Step 是阶段中的最小执行单元
type Step struct {
    ID         string                 `json:"id"`
    Type       StepType              `json:"type"`
    Name       string                 `json:"name"`
    Spec       map[string]interface{} `json:"spec"`
    RetryCount int                    `json:"retry_count"`
    RetryDelay time.Duration          `json:"retry_delay"`
}
```

**执行状态机**：

```
StateCreated → StateValidating → StateQueued → StateRunning → StatePaused
                                                                    |
                                              StateCompleted       StateFailed
                                              StateCancelled       StateExpired
```

**并行执行调度器**：
- 构建阶段依赖关系图（DAG）
- 拓扑排序确定执行顺序
- 信号量控制并发度（maxConcurrency）
- 工作协程池执行各阶段

---

## 四、CI/CD实现方式

### 4.1 声明式Pipeline as Code（YAML）

```yaml
pipeline:
  name: "microservice-cd"
  identifier: "ms_cd"
  stages:
    - stage:
        name: "Build"
        type: "CI"
        spec:
          cloneCodebase: true
          platform:
            os: "Linux"
            arch: "Amd64"
          execution:
            steps:
              - step:
                  type: "Run"
                  name: "Build Image"
                  identifier: "build_image"
                  spec:
                    shell: "Sh"
                    command: |
                      docker build -t myapp:${{ variables.BUILD_NUMBER }} .
    - stage:
        name: "Deploy to Prod"
        type: "CD"
        spec:
          infrastructure:
            environmentRef: "prod-env"
            infrastructureDefinition:
              type: "KubernetesDirect"
          execution:
            steps:
              - step:
                  type: "K8sRollingDeploy"
                  name: "Rolling Deployment"
                  identifier: "rolling_deploy"
```

### 4.2 高级部署策略（内置，无需自定义脚本）

| 策略 | 原理 | 适用场景 | 回滚速度 |
|---|---|---|---|
| **滚动部署** | 逐步替换旧Pod为新Pod | 无状态应用、内部工具 | 中等 |
| **蓝绿部署** | 同时运行两套环境，切换流量 | 核心交易系统 | 秒级 |
| **金丝雀部署** | 先升级少量Pod逐步扩大 | 高风险功能 | 快 |

Harness提供**自动金丝雀分析**，集成Prometheus、Datadog、New Relic等，失败自动回滚。

### 4.3 与Jenkins、GitHub Actions的核心差异

| 维度 | Jenkins | GitHub Actions | Harness |
|---|---|---|---|
| **架构理念** | 插件化自动化服务器 | GitHub原生CI/CD | AI原生软件交付平台 |
| **配置方式** | Groovy脚本 | YAML配置 | 声明式YAML+可视化编辑器 |
| **部署策略** | 需自定义脚本 | 基础策略 | 内置高级策略（金丝雀/蓝绿） |
| **AI能力** | 无原生AI | Copilot集成较深 | 内置AIDA贯穿全流程 |
| **运维成本** | 高（需2-5人专职维护） | 低（云端托管） | 极低（SaaS免运维） |
| **安全治理** | 需插件组合 | 企业版支持 | 内置RBAC+OPA策略引擎 |
| **学习曲线** | 陡峭 | 极低 | 中（可视化+AI辅助） |
| **云原生** | 传统Java架构 | 良好 | 原生容器化架构 |

---

## 五、核心技术栈和开源组件

| 层级 | 技术 | 用途 |
|---|---|---|
| **编程语言** | **Go**（58%）、**TypeScript**（36%） | 后端核心+前端UI |
| **通信协议** | gRPC、REST API、WebSocket | 微服务间通信、Delegate连接 |
| **数据库** | PostgreSQL、MongoDB、Redis | 元数据、非结构化数据、缓存 |
| **消息队列** | NATS、RabbitMQ | 事件驱动架构 |
| **容器编排** | Kubernetes、Docker | 基础设施运行 |
| **UI框架** | React + TypeScript | Web界面 |
| **CLI** | Go + Kingpin | 命令行工具 |
| **API文档** | Swagger/OpenAPI | 自动生成API文档 |
| **策略引擎** | OPA (Open Policy Agent) | 策略即代码 |
| **许可协议** | Apache License 2.0 | 开源 |

### AI原生能力：AIDA + 知识图谱

**AIDA（AI Development Assistant）**：
- 自动生成Pipeline YAML
- 修复构建失败，分析日志根因
- 安全漏洞自动修复（基于CVE/CWE训练）
- 云成本自然语言管理

**Software Delivery Knowledge Graph**：
- 构建专用查询语言HQL（Harness Query Language）
- 相比直接调用API，**Token成本降低15-25倍**（从25-35万token降至约1.2万token）
- 输出确定、无幻觉

---

## 六、GitHub 开源项目

### 6.1 官方组织

地址：**[github.com/harness](https://github.com/harness)** （169个公开仓库）

### 6.2 关键仓库一览

| 仓库 | Stars | 语言 | 描述 |
|---|---|---|---|
| [**harness/harness**](https://github.com/harness/harness) | **36,000+** | Go+TS | Harness Open Source -- 端到端开发者平台 |
| [**harness/gitness**](https://github.com/harness/gitness) | **32,100+** | Go+TS | Gitness -- 前身项目，轻量级DevOps平台 |
| [**harness/drone-cli**](https://github.com/harness/drone-cli) | 433 | Go | Drone CI命令行工具 |
| [**harness/harness-cd-community**](https://github.com/harness/harness-cd-community) | 211 | Shell | Harness CD社区版（已归档） |
| [**harness/mcp-server**](https://github.com/harness/mcp-server) | 59 | TS | 官方MCP Server，供AI Agent交互 |
| [**harness/terraform-provider-harness**](https://github.com/harness/terraform-provider-harness) | 45 | Go | Terraform Provider |
| [**harness/developer-hub**](https://github.com/harness/developer-hub) | 80 | HTML | 官方文档门户 |
| [**harness/canary**](https://github.com/harness/canary) | 15 | TS | 下一代统一UI（Monorepo） |
| [**harness/harness-cli**](https://github.com/harness/harness-cli) | 18 | Go | Harness CLI工具 |
| [**harness/harness-go-sdk**](https://github.com/harness/harness-go-sdk) | 16 | Go | Harness API Go SDK |
| [**harness/ff-golang-server-sdk**](https://github.com/harness/ff-golang-server-sdk) | 11 | Go | Feature Flags服务端SDK |

### 6.3 项目演进历史

**Drone → Gitness → Harness**

1. **Drone**：Harness公司于2020年收购的容器原生CI平台（github.com/drone/drone），专注持续集成
2. **Gitness**：基于Drone构建的开源开发者平台，增加了源码托管（类GitLab）
3. **Harness Open Source**：Gitness的进一步演进，增加了Gitspaces和Artifact Registries

---

## 七、部署形态

| 形态 | 说明 | 适用场景 |
|---|---|---|
| **SaaS（云托管）** | Harness托管的SaaS服务 | 多数企业，免运维 |
| **Self-Managed Enterprise** | 自托管企业版 | 严格合规要求 |
| **Community/Open Source** | 社区版（Apache 2.0） | 学习、小团队 |

快速启动：

```bash
docker run -d \
  -p 3000:3000 \
  -p 3022:3022 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/harness:/data \
  --name harness \
  --restart always \
  harness/harness
```

---

## 八、总结

Harness平台的核心架构特点：

1. **架构范式**：从"自动化服务器"（Jenkins）到"平台化软件交付"，覆盖SDLC全生命周期
2. **Delegate机制**：通过出站WebSocket连接的轻量级代理，实现无入站端口的安全执行层
3. **声明式Pipeline as Code**：纯YAML定义，Git版本化，内置模板/变量/并行/条件执行
4. **事件驱动微服务架构**：gRPC+REST+NATS/RabbitMQ，PostgreSQL+MongoDB+Redis
5. **AI原生集成**：AIDA助手贯穿CI/CD/Security/Cost全流程，知识图谱大幅降低推理成本
6. **技术栈**：Go后端+TypeScript/React前端+Kubernetes/Docker运行时，Apache 2.0开源

---

## 九、参考资源

- [Harness官方GitHub](https://github.com/harness)
- [Harness Open Source主仓库](https://github.com/harness/harness) （36k+ Stars）
- [Gitness仓库](https://github.com/harness/gitness) （32k+ Stars）
- [Drone CLI仓库](https://github.com/harness/drone-cli)
- [Harness MCP Server](https://github.com/harness/mcp-server)
- [官方文档](https://developer.harness.io)
- [CSDN架构深度解析](https://blog.csdn.net/m0_50709695/article/details/159773586)
- [CSDN平台全景对比分析](https://blog.csdn.net/RickyIT/article/details/161359414)
- [CSDN CI/CD+GitOps实战](https://blog.csdn.net/RickyIT/article/details/161359435)
- [腾讯云Gitness源码剖析](https://cloud.tencent.com/developer/article/2357246)
- [腾讯云Harness开发平台介绍](https://cloud.tencent.com/developer/article/2689874)

---

*本报告基于2026年6月公开资料编写，持续更新中...*
