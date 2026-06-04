ragflow-freecad-heat-exchanger



🧠 AI 驱动的管壳式换热器智能设计系统

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

> 告别反复查表与手动建模！输入设计需求，AI 自动检索国标规范、完成热力计算与校核，最后调用 FreeCAD 生成三维模型。

## 📌 项目动机

传统换热器设计流程中，工程师需要：
- 查阅 GB/T 151 等标准获取物性、结构参数
- 手工进行多次迭代计算（传热系数、压降）
- 使用 CAD 软件手动绘制三维模型

本项目将 **RAGFlow 知识库**、**Claude Code 智能体** 与 **FreeCAD** 集成，实现：
> 自然语言需求 → 规范检索 + 工作流计算 → 结构校核 → 参数化三维建模
**实现快速出图**

## 🏗️ 系统架构

```
用户输入设计条件（介质、温度、流量...）
         │
         ▼
┌─────────────────────────────────┐
│       Claude Code (Agent)       │
│  调度 MCP 工具，遵循四步流程      │
└───────┬──────────┬──────────────┘
        │          │
        ▼          ▼
┌───────────┐  ┌─────────────────┐
│ RAGFlow   │  │ FreeCAD (WSL)   │
│ 知识库     │  │ 参数化建模脚本    │
│ GB/T 151  │  │ build_hx_v2.py  │
│ 工作流 JSON│  │ fc_daemon       │
└───────────┘  └─────────────────┘
        │          │
        ▼          ▼
   检索规范      生成 STL/STEP
   设计计算      三维装配体
   机械校核
```

### 🔗 关键技术栈

| 组件 | 作用 |
|------|------|
| **RAGFlow** | 存放《GB/T 151-2014 热交换器》等标准文档，并提供工作流实现设计计算与规范核验 |
| **Claude Code** | 作为智能 Agent，理解用户需求，调度工具，严格遵循四步设计流程 |
| **MCP (FastMCP)** | 将 Python 函数封装为 Claude Code 可调用的工具（检索知识库、触发建模） |
| **FreeCAD + WSL** | 在无 GUI 的 Linux 环境中运行 Python 脚本，生成换热器三维模型 |
| **Docker** | 容器化部署 RAGFlow，确保环境一致性 |

## 💡 核心创新点

### 1. 分割设计零件，分步建模
为了避免一次性生成全模型时参数耦合导致错误，我们采用**逐部件建模策略**：
- 将换热器拆解为壳体、管板、换热管、折流板、管嘴、法兰、鞍座等独立零件
- 每个零件使用固定的辅助函数 (`cyl_x`, `cyl_z`, `Box`) 创建
- 用 `BoundingBox` 实时验证位置，最后通过 `MultiFuse` 融合

### 2. 知识库驱动的规范核验
- RAGFlow 导入 GB/T 151 全文及常用设计数据表
- 设计过程中，Agent 调用 `search_knowledge_base` 检索标准条文，自动检查管板外径、折流板间隙等是否符合规范
- 发现冲突立即暂停，要求用户确认或修正

### 3. 内置“错误记忆库”
在长期调试中积累了 **11 类常见建模错误**（如后管板台阶错位、鞍座 Box 方向颠倒、封头角度诡异等），我们将其整理为**历史错误数据库**，并写入 Agent 的提示词中。这使得后续设计时，Agent 能**主动规避**这些已知雷区。

## 📁 仓库结构

```
.
├── README.md                   # 本文件
├── LICENSE
├── images/                     # 架构图、模型截图、流程演示
│   ├── architecture.png
│   └── model_preview.png
├── prompts/
│   └── design_workflow.md      # Claude Code 遵循的详细设计流程提示词
├── mcp-servers/
│   ├── ragflow_server.py       # 检索知识库的 MCP 工具
│   └── freecad_server.py       # 调用 WSL 中 FreeCAD 的 MCP 工具
├── freecad/
│   ├── build_hx_v2.py          # 换热器参数化建模主脚本
│   └── fc_daemon_fixed.py      # FreeCAD HTTP 守护进程（可选）
├── ragflow/
│   └── agent_12121.json        # RAGFlow 工作流（问题分类→参数收集→设计→核算→机械设计）
└── examples/
    └── design_example.md       # 一次完整的设计对话记录
```

## 🚀 快速开始

### 环境要求
- **Windows 10/11** + WSL2 (Ubuntu 22.04)
- **Docker Desktop** (运行 RAGFlow)
- **Python 3.10+** (Windows 侧)
- **Claude Code** 客户端
- **FreeCAD 0.19+** (安装在 WSL 内)

### 1. 克隆仓库
```bash
git clone https://github.com/你的用户名/ragflow-freecad-heat-exchanger.git
cd ragflow-freecad-heat-exchanger
```

### 2. 启动 RAGFlow 并导入知识库
```bash
cd ragflow
docker compose up -d
```
- 访问 `http://localhost` 进入 RAGFlow 管理界面
- 创建知识库，上传 `GB_T_151-2014.pdf` 等规范文件
- 导入工作流 `agent_12121.json`（可选）

### 3. 配置 WSL 环境
在 WSL 终端中：
```bash
sudo apt update
sudo apt install freecad -y
# 将 freecad/ 下的脚本复制到 WSL 中
cp /mnt/c/你的仓库路径/freecad/* ~/fc_daemon/
```

### 4. 安装 Python 依赖（Windows 侧）
```powershell
pip install fastmcp requests
```

### 5. 配置 Claude Code MCP 服务器
编辑 `C:\Users\你的用户名\.mcp.json`：
```json
{
  "mcpServers": {
    "ragflow-search": {
      "command": "python",
      "args": ["C:\\你的仓库路径\\mcp-servers\\ragflow_server.py"]
    },
    "freecad-designer": {
      "command": "python",
      "args": ["C:\\你的仓库路径\\mcp-servers\\freecad_server.py"]
    }
  }
}
```

### 6. 启动设计
打开 Claude Code，输入 `/mcp` 确认工具已连接，然后说：
```
/exchanger 设计一台固定管板式换热器，管程对二甲苯 15 m³/h，
80℃→45℃，壳程冷却水 30℃→39℃，设计压力 1.0 MPa。
```
Agent 将按照 **计算→零件清单→校核→建模** 四步流程执行，最终在 WSL 的 `/tmp/` 下生成 `heat_exchanger.stl`。

## 🎬 演示


## 🧠 提示词工程

本项目的核心是 Claude Code 遵循的详细设计流程提示词（见 `prompts/design_workflow.md`），它包含了：
- 严格的四步工作流（计算→清单→校核→建模）
- 坐标系铁律与辅助函数规范
- 基于真实 bug 的错误数据库（避免重蹈覆辙）
- 每步完成后的验证机制

你可以直接复制该提示词作为自定义命令 `/exchanger` 使用。

## 📈 改进方向

- [ ] 支持导出 STEP 装配体，保留颜色与材质
- [ ] 增加 U 型管、浮头式换热器模板
- [ ] 集成 GB/T 150 压力容器强度计算
- [ ] 使用 CCSwitch 图形化管理 MCP 配置
- [ ] 前端界面，无需命令行操作

## 📄 许可证

MIT License


## 项目思路与创新点
1. 从「一步到位」到「分而治之」
最初我试图让 AI 一次性输出完整的 FreeCAD 建模脚本，但 3000 多 token 的尝试都以混乱失败告终——部件相互耦合、坐标错乱，训练出来的模型面对复杂装配体也基本无效。
于是我转变策略：将整台换热器拆解为可独立管理的零件单元——壳体、管板、换热管、折流板、管嘴、法兰、鞍座等——每个零件用专用函数创建，最后再用布尔运算融合成整体。这种“分步设计、最后总装”的方法大幅提升了模型生成的成功率和可调试性。

2. 严格四步设计流程
为了避免 AI 跳过关键步骤，我设计了强约束的 计算→零件清单→校核→建模 四步工作流，并内嵌到提示词中：

第一步：完成热力与结构设计计算，输出标准化参数表，锁定所有关键尺寸。

第二步：生成结构化零部件清单，明确每件的配合关系与生成方法。

第三步：依据国标进行配合自洽性与标准符合性核验，不通过则暂停修正。

第四步：调用 FreeCAD 逐部件建模并融合导出。

3. 知识库驱动的规范校核
我将先前在 RAGFlow 中配置的工作流 JSON 直接接入系统，同时将《GB/T 151-2014 热交换器》等规范文件导入知识库。
设计过程中，AI 可随时检索标准条文对参数进行验证，例如自动检查管板外径与壳体配合间隙、折流板圆缺高度等——这层“规范守门人”让自动化设计有了工程安全感。

4. 解决 WSL 下 FreeCAD 的稳定性难题
在 WSL 无 GUI 环境中直接调用 FreeCAD 经常崩溃或僵死。为此我编写了守护进程与保护代码，通过 HTTP 接口接收任务、超时重启、异常回退，保证了批量建模的可靠性。同时封装了 cyl_x、cyl_z 等坐标系辅助函数，彻底消除因 FreeCAD 旋转 API 误用导致的 90% 模型 bug。

5. 封装为可复用的 Skill
整套流程——从对话触发、知识检索、计算校核到建模导出——最终被封装成一个 Claude Code 的自定义命令 /exchanger。他人只需克隆仓库、启动 RAGFlow 和 WSL 环境，即可通过一句自然语言启动换热器设计，真正做到了“下载即用”。


## 🙏 致谢

- [RAGFlow](https://github.com/infiniflow/ragflow) - 深度文档理解与检索
- [FreeCAD](https://www.freecad.org/) - 开源参数化三维建模
- [Claude Code](https://claude.ai/code) - AI 编程助手
- [FastMCP](https://github.com/jlowin/fastmcp) - 轻松构建 MCP 服务器
```

