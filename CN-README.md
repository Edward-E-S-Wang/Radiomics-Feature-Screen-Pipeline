# 放射组学特征筛选全流程（Radiomics Feature Screen Pipeline）

这是一个面向**二分类任务**的完整放射组学特征筛选流程，整合了 **Mann–Whitney U 检验**、**Spearman 相关性分析**和 **mRMR（最小冗余最大相关性）** 三种方法，用于逐步降低特征维度并保留最具信息量的特征。

该流程专为结构化放射组学数据设计，尤其适用于医学影像研究场景。在这类研究中，手工提取的高维特征数量通常远大于样本量，因此有必要进行系统性的特征筛选，以提高模型的稳定性和可解释性。

---

## 概述

高维放射组学特征中通常包含大量**无关特征、不稳定特征或冗余特征**。如果将全部提取特征直接用于模型构建，可能会导致以下问题：

- 过拟合
- 模型性能不稳定
- 可解释性差
- 计算成本增加

为解决这些问题，本项目实现了一套**三阶段特征筛选流程**：

1. **Mann–Whitney U 检验**：用于初步单因素筛选  
2. **Spearman 相关性分析**：用于去除冗余特征  
3. **mRMR**：用于最终的多变量特征排序与选择  

最终目标是获得一组**紧凑、稳健且具有较强判别能力**的特征子集，用于后续建模分析。

---

## 工作流程

完整流程如下：

**数据读取 → 缺失值处理 → Mann–Whitney U 初筛 → Spearman 冗余去除 → mRMR 精筛 → 导出最终数据集**

---

## 筛选策略

## 第一步：数据读取与预处理

脚本读取输入的 CSV 文件，并默认数据结构如下：

- **第1列**：患者 ID 或病例编号  
- **第2列**：二分类标签  
- **第3列及之后**：放射组学特征  

在预处理过程中，程序将执行以下操作：

- 将标签列转换为数值型
- 将所有特征列转换为数值型
- 无法转换的值自动记为 `NaN`
- 标签列中的缺失值使用**均值填补**
- 特征列中的缺失值使用**各列均值填补**

该步骤的目的是保证后续统计分析不会因非数值项或缺失值而中断。

---

## 第二步：Mann–Whitney U 检验

第一阶段使用 **Mann–Whitney U 检验** 来识别在两组之间存在显著差异的特征。

### 原理说明

放射组学特征往往不服从正态分布。与 t 检验相比，Mann–Whitney U 检验是一种**非参数检验方法**，在不满足正态分布假设时更加适用。

### 具体过程

对于每一个特征：

- 分别提取 0 类和 1 类样本中的该特征值
- 执行双侧 Mann–Whitney U 检验
- 记录其对应的 `p` 值

仅保留满足以下条件的特征进入下一步：

```text
p < 0.05
````

### 目的

该步骤用于剔除与结局变量关系不明显的特征，是一个初步的统计学筛选过程。

### 输出文件

* `01_MWU_p_values.csv`
  保存所有原始特征对应的 Mann–Whitney U 检验 `p` 值

* `02_MWU_significant_features.csv`
  保存通过 Mann–Whitney U 检验后保留下来的显著特征

---

## 第三步：Spearman 相关性分析

经过单因素筛选后，保留下来的特征之间仍可能存在较强相关性。为进一步减少冗余信息，第二阶段采用 **Spearman 相关性分析**。

### 原理说明

高度相关的放射组学特征往往携带重复或相近的信息。如果全部保留，可能会：

* 增加共线性
* 降低模型稳定性
* 降低模型可解释性

由于 Spearman 相关系数是基于秩次的统计量，因此对非正态数据更加稳健，适合放射组学数据分析。

### 具体过程

* 计算所有保留特征两两之间的 **Spearman 绝对相关系数**
* 如果任意两个特征之间满足：

```text
|ρ| > 0.90
```

则认为这两个特征存在较强冗余，仅保留其中一个，删除另一个。

### 删除规则

当两个特征高度相关时，流程会优先保留 **Mann–Whitney U 检验中 p 值更小** 的特征，因为该特征在上一阶段中表现出更强的单因素区分能力。

### 目的

该步骤用于去除冗余特征，在进入多变量筛选之前尽量保留信息多样性更高的特征集合。

### 输出文件

* `03_Spearman_correlation_matrix.csv`
  保留特征之间的绝对 Spearman 相关矩阵

* `04_Spearman_dropped_features.csv`
  记录冗余特征删除详情，包括保留了哪个特征、删除了哪个特征及其相关系数

* `05_Features_after_Spearman.csv`
  Spearman 去冗余后剩余的特征矩阵

---

## 第四步：mRMR 特征选择

最后一阶段使用 **mRMR（minimum Redundancy Maximum Relevance，最小冗余最大相关性）** 方法，对特征进行最终排序和筛选。

### 原理说明

即使经过显著性筛选和冗余去除，剩余特征数量仍可能较多，不利于构建稳健模型。mRMR 的目标是筛选出：

* 与结局变量**高度相关**
* 与已选特征之间**冗余尽可能小**

的一组最优特征。因此，mRMR 特别适合高维生物医学数据和放射组学研究。

### 实现方式

本项目实现的是一个**不依赖 `pymrmr` 的自包含 mRMR 算法**。

主要步骤包括：

1. 使用**中位数**填补剩余缺失值
2. 将连续特征进行**分位数离散化**
3. 计算互信息：

   * **相关性（Relevance）**：特征与标签之间的互信息
   * **冗余性（Redundancy）**：候选特征与已选特征之间的平均互信息
4. 采用**贪婪前向搜索策略**，逐步选出目标数量的特征

### 支持的准则

本流程支持两种 mRMR 评分方式：

* **MID**
  `score = relevance - redundancy`

* **MIQ**
  `score = relevance / (redundancy + eps)`

默认使用：

```text
MID
```

### 最终特征数

默认选择前：

```text
TOP_K = 50
```

个特征。如果候选特征总数不足 50，则保留全部候选特征。

### 目的

该步骤用于生成最终用于建模的紧凑型特征子集。

### 输出文件

* `06_mRMR_relevance.csv`
  各特征与标签之间的互信息相关性结果

* `07_mRMR_selected_features.txt`
  最终被选中的特征名称列表

* `08_mRMR_selection_process.csv`
  mRMR 每一步选择过程的详细记录，包括相关性、冗余度和最终得分

* `09_mRMR_selected_feature_values.csv`
  仅包含 mRMR 最终保留特征的特征矩阵

---

## 第五步：最终数据集导出

完成 mRMR 筛选后，脚本会生成最终建模数据集，其内容由以下两部分拼接而成：

* 原始数据中的前两列
* mRMR 最终保留的特征子集

### 输出文件

* `10_Final_selected_dataset.csv`
  用于后续机器学习或统计建模的最终数据集

* `11_feature_count_summary.csv`
  各个筛选阶段剩余特征数量的汇总表

---

## 为什么采用三阶段筛选

该流程的设计目标是同时兼顾：

* **统计显著性**
* **冗余控制**
* **多变量相关性优化**

### Mann–Whitney U 检验

用于剔除在组间无显著差异的特征。

### Spearman 相关性分析

用于去除高度相关的冗余特征，降低共线性和不稳定性。

### mRMR

用于在剩余特征中进一步筛选出与标签最相关、同时彼此冗余最小的特征集合。

三者结合，相比单独使用任意一种方法，更能构建出稳健、可解释性更强的特征筛选框架。

---

## 输入文件要求

输入文件必须为 CSV 格式。

推荐数据结构如下：

| 列号     | 内容说明         |
| ------ | ------------ |
| 第1列    | 患者 ID / 病例编号 |
| 第2列    | 二分类标签        |
| 第3列及以后 | 放射组学特征       |

示例：

```text
ID,Label,feature_1,feature_2,feature_3,...
001,0,0.123,4.567,8.910,...
002,1,0.234,3.456,7.890,...
```

### 注意事项

* 标签列必须是**二分类任务**
* 所有特征列应为数值型，或至少可以转换为数值型
* 非数值项将被自动转换为缺失值并进行填补
* 如果你的数据结构与默认设置不同，请修改脚本中的参数：

```python
LABEL_COL_INDEX = 1
FEATURE_START_COL_INDEX = 2
```

例如，如果第一列就是标签且没有 ID 列，则可改为：

```python
LABEL_COL_INDEX = 0
FEATURE_START_COL_INDEX = 1
```

---

## 默认参数

```python
MWU_P_THRESHOLD = 0.05
SPEARMAN_THRESHOLD = 0.90
TOP_K = 50
MRMR_CRITERION = "MID"
N_BINS = 10
```

### 参数说明

* `MWU_P_THRESHOLD`
  Mann–Whitney U 检验显著性阈值

* `SPEARMAN_THRESHOLD`
  Spearman 绝对相关系数冗余判定阈值

* `TOP_K`
  mRMR 最终选择的特征数量

* `MRMR_CRITERION`
  mRMR 评分方式，可选 `MID` 或 `MIQ`

* `N_BINS`
  计算互信息前连续特征离散化时的分箱数量

---

## 输出文件说明

运行流程后，输出目录中会生成以下文件：

| 文件名                                   | 说明                          |
| ------------------------------------- | --------------------------- |
| `01_MWU_p_values.csv`                 | 所有特征的 Mann–Whitney U 检验 p 值 |
| `02_MWU_significant_features.csv`     | MWU 显著特征列表                  |
| `03_Spearman_correlation_matrix.csv`  | Spearman 绝对相关矩阵             |
| `04_Spearman_dropped_features.csv`    | 冗余特征删除记录                    |
| `05_Features_after_Spearman.csv`      | Spearman 去冗余后特征矩阵           |
| `06_mRMR_relevance.csv`               | 特征与标签之间的互信息相关性              |
| `07_mRMR_selected_features.txt`       | 最终入选特征名称                    |
| `08_mRMR_selection_process.csv`       | mRMR 选择过程详情                 |
| `09_mRMR_selected_feature_values.csv` | 最终 mRMR 特征矩阵                |
| `10_Final_selected_dataset.csv`       | 最终建模数据集                     |
| `11_feature_count_summary.csv`        | 各阶段特征数汇总                    |

---

## 流程示例解释

假设原始数据集中共有 **1200 个放射组学特征**，则一个典型筛选过程可能如下：

* 原始特征数：**1200**
* Mann–Whitney U 初筛后：**180**
* Spearman 去冗余后：**74**
* mRMR 最终筛选后：**50**

这意味着：

* 首先去除了与结局无明显关联的特征
* 然后删除了高度相关的冗余特征
* 最终保留了一组兼具相关性和互补性的特征子集

---

## 本流程的优势

* 适用于高维放射组学数据
* 同时结合单因素和多因素特征筛选思想
* 在高级排序前先进行冗余控制
* 有助于提高模型可解释性和重复性
* 不依赖外部 `pymrmr`
* 易于迁移到其他二分类表格数据任务中

---

## 局限性

* 当前实现仅适用于**二分类任务**
* 当前形式下的 Mann–Whitney U 检验不适用于多分类标签
* mRMR 中的离散化可能带来一定信息损失
* 不同研究中阈值设置可能需要根据样本量和设计进行调整

---

## 推荐应用场景

该流程适用于以下任务：

* 基于放射组学的诊断模型构建
* 预后预测研究
* 治疗反应分类
* 机器学习前的高维生物医学特征降维
* 其他涉及高维表格特征的二分类任务

---

## 未来可扩展方向

后续可以考虑增加以下功能：

* 支持多分类任务
* 支持回归任务
* 在 Mann–Whitney U 检验后加入多重比较校正
* 引入基于交叉验证的稳定特征筛选策略
* 与 LASSO、SVM、Random Forest、XGBoost 等下游模型进一步整合

---

## Methods 描述示例

如果你在研究中使用本流程，建议在论文 Methods 部分明确描述特征筛选策略，包括：

* 基于 Mann–Whitney U 检验的非参数单因素筛选
* 基于 Spearman 相关性分析的冗余特征去除
* 基于 mRMR 的最终特征选择

一个可直接参考的英文写法为：

> Features were first screened using the Mann–Whitney U test, and those with p < 0.05 were retained. Redundant features were then removed using Spearman correlation analysis with a threshold of |ρ| > 0.90, keeping the feature with the smaller p-value when a highly correlated pair was identified. Finally, mRMR was applied to select the most relevant and least redundant features for model development.

对应中文可表述为：

> 首先采用 Mann–Whitney U 检验对特征进行初步筛选，保留 p < 0.05 的特征。随后基于 Spearman 相关性分析去除冗余特征，当两特征的绝对相关系数大于 0.90 时，保留 p 值更小的特征。最后采用 mRMR 方法筛选出与标签最相关且冗余最小的特征，用于后续模型构建。

---

## 引用说明

如果你在学术研究中使用了本代码、该实现策略或其修改版本，请引用以下原始文章：

Wang, Y., Zhang, H., Wang, H. et al. Development of a neoadjuvant chemotherapy efficacy prediction model for nasopharyngeal carcinoma integrating magnetic resonance radiomics and pathomics: a multi-center retrospective study. *BMC Cancer* 24, 1501 (2024). [https://doi.org/10.1186/s12885-024-13235-0](https://doi.org/10.1186/s12885-024-13235-0)

