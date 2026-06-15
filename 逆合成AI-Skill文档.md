# 🧪 DongLab 逆合成 AI Skill

> 为 AI Agent 提供逆合成规划能力的 Skill 文档  
> 版本: v1.1 | 维护: 张一健 (Mr.自由基) | 实验室: DongLab, AI4S 课题组

---

## 什么是 AI Skill？

AI Skill 是 AI Agent（如 Claude、OpenClaw 等）可加载的功能模块。本文档描述了 DongLab 逆合成规划系统的 Skill 接口，供开发者将其集成到自己的 AI 助手中使用。

## 使用方法

### 给 AI Agent 安装

将本 Skill 配置到 AI Agent 的技能目录下（如 OpenClaw 的 `skills/` 目录），或在 Agent 提示词中引用以下 API 说明。

### 快速测试

```bash
# 单步预测（gu03，推荐）
curl -X POST http://114.212.161.184:8001/retroplanner/api/single_step \
  -H "Content-Type: application/json" \
  -d '{"smiles":"CC(=O)Oc1ccccc1C(=O)O","savedOptions":{"topk":10,"oneStepModel":["Pistachio"]}}'

# 备选服务器（gu26）
curl -X POST http://114.212.160.159:8001/retroplanner/api/single_step \
  -H "Content-Type: application/json" \
  -d '{"smiles":"CC(=O)Oc1ccccc1C(=O)O","savedOptions":{"topk":10,"oneStepModel":["Pistachio"]}}'
```

## 服务器拓扑

| 服务器 | IP | 主要服务 | 状态 |
|:------|:---|:--------|:----:|
| gu03 | 114.212.161.184:8001 | 主站 API（9 模型 + 多步 + 酶） | OK |
| gu26 | 114.212.160.159:8001 | 同上（备选，同镜像） | OK |
| gu26 | 114.212.160.159:5000 | Retro* 多步规划 | OK |

gu03 与 gu26 互为冗余，API 完全相同，任一不可用时自动切换。

## 可用模型（9 个）

| 模型 | 类型 | 数据源 | 速度 |
|:----|:-----|:-------|:----:|
| Pistachio | 模板匹配 | 专利数据库 | <0.1s |
| Reaxys | 模板匹配 | Reaxys 文献 | <0.1s |
| GraphFP (USPTO-Full) | 图神经网络 | USPTO-Full | ~0.5s |
| RxNGormer (USPTO-Full) | 图Transformer | USPTO-Full | ~2-3s |
| ChemBart (Full) | BART 语言模型 | USPTO-Full | ~1-3s |
| Transformer | Transformer | USPTO-NPL+BioChem | ~1-2s |
| Pistachio Ringbreaker | 模板匹配 | 开环反应 | <0.1s |
| Reaxys Biocatalysis | 模板匹配 | 生物催化 | <0.1s |
| BKMS Metabolic | 模板匹配 | 代谢反应 | <0.1s |

## API 说明

### 1. 单步逆合成预测

```bash
POST /retroplanner/api/single_step
参数：
- smiles (string, 必填): 目标分子 SMILES
- topk (int, 可选): 返回前 K 个结果，默认 10
- oneStepModel (string[]): 模型选择，如 ["Pistachio", "RxNGormer"]

curl -X POST http://114.212.161.184:8001/retroplanner/api/single_step \
  -H "Content-Type: application/json" \
  -d '{"smiles":"CC(=O)Oc1ccccc1C(=O)O","savedOptions":{"topk":10,"oneStepModel":["Pistachio","RxNGormer"]}}'
```

### 2. 多步逆合成规划（MCTS_STAR）

```bash
POST /retroplanner/api/retroplanner
参数：
- smiles (string, 必填): 目标分子 SMILES
- iterations (int, 可选): 搜索迭代次数，默认 10
- max_depth (int, 可选): 最大深度，默认 6

curl -X POST http://114.212.161.184:8001/retroplanner/api/retroplanner \
  -H "Content-Type: application/json" \
  -d '{"smiles":"CC(=O)Oc1ccccc1C(=O)O","savedOptions":{"topk":10,"iterations":10,"max_depth":6}}'
```

### 3. Retro* 多步规划（gu26:5000）

```bash
POST /plan
curl -X POST http://114.212.160.159:5000/plan \
  -H "Content-Type: application/json" \
  -d '{"smiles":"CC(=O)Oc1ccccc1C(=O)O"}'
```

### 4. 辅助功能（gu03）

| 功能 | 端点 | 说明 |
|:----|:-----|:-----|
| 反应条件预测 | /api/condition_predictor | 温度、溶剂、催化剂 |
| 反应可行性 | /api/reaction_rater | 可行性评分 |
| 酶催化识别 | /api/enzymatic_rxn_identifier | 是否酶催反应 |
| 酶推荐 | /api/enzyme_recommender | EC 编号推荐 |
| 酶活性位点 | /api/easifa | ESM+GNN 结构分析 |

### 5. 分子渲染

```bash
POST http://119.45.174.234/retro/api/render_svg
参数：{"smiles": "CC(=O)Oc1ccccc1"}
返回：{"svg": "..."}
```

或使用 RDKit 本地渲染：

```python
from rdkit import Chem
from rdkit.Chem import Draw
mol = Chem.MolFromSmiles("CCO")
svg = Draw.MolsToGridImage([mol], molsPerRow=1, subImgSize=(250,150), useSVG=True)
```

## 集成到 AI Agent

将本文档作为 Skill 文件放入 AI 的技能目录，AI 即可通过调用上述 API 实现逆合成规划能力。

### 核心能力

| 能力 | 触发场景 |
|:----|:---------|
| 单步预测 | 用户给一个分子询问「怎么合成」|
| 多步规划 | 用户要求「规划完整合成路线」|
| 条件预测 | 用户问「什么条件做这个反应」|
| 酶催化分析 | 用户问「这个反应有酶能催化吗」|

## 已知问题

| 模型 | 问题 |
|:----|:-----|
| RxNGormer | 50K 版已弃用，仅保留 Full 版，可能返回空 |
| Pistachio Ringbreaker | 无环分子返回空 |
| Agent LLM | 多步规划 Agent 需外网访问 LLM API |

## 参考

- 主文献: Wang, X. et al. Nat. Commun. 2025, 16, 10929
- 项目主页: ChemMixPlanner (gu26:8001)
- 维护: 张一健 (zhangyijian@qq.com)
