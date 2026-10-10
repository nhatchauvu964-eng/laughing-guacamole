# 热轧板带数据预处理与半监督回归

面向热轧板带生产数据的 Python / Jupyter Notebook 实验项目，以 `RTT_RPDOT2`（常温拉伸屈服强度）为默认回归目标，串联数据清洗、时间与炉号隔离、KNN 密集样本裁剪和随机森林回归自训练。

项目保留有标签与无标签样本的处理记录，便于检查样本去向和伪标签来源。核心函数均内嵌在 Notebook 中，无需配套 Python 模块。

> 仓库不包含原始生产数据。运行前需要准备符合下述字段要求、具有使用权限的 CSV。现有导出结果也不作为公开示例发布。

## 功能与处理流程

1. **按时间和炉号划分**：切分日期之前的整炉数据用于训练，切分日期及之后的整炉数据用于测试；跨越切点的炉次单独隔离。无效日期或缺失炉号另行保存。
2. **区分标签状态**：目标列非空的样本作为有标签数据，目标为空的样本单独保留，预处理不填充目标。
3. **特征清洗**：仅用有标签训练数据选择数值特征、筛除高缺失率列、拟合中位数和确定缺失标记；各分区分别隔离严重缺失行。
4. **KNN 密集样本裁剪**：对有标签训练特征标准化后计算平均近邻距离，从最密集的候选样本中随机保留一部分。标准化用于距离计算，输出特征仍保留原尺度。
5. **回归自训练**：在训练时段内再次按时间和炉号划分内部验证集，并重新拟合内部预处理。通过随机森林各棵树的预测分歧筛选伪标签，用较低权重加入训练。
6. **选择与评估**：按内部验证 MAE 选择候选轮数；若未优于纯监督基线，推荐模型回退基线。最终测试集只用于报告 MAE、RMSE 和 R²。
7. **审计与导出**：导出增强训练集、伪标签审计表、未采用的无标签样本、内部验证记录和测试指标。

## 项目结构

```text
.
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
├── docs/
│   └── GITHUB_PUBLISHING.md
└── 数据预处理/
    ├── 数据预处理.ipynb
    ├── data/                       # 自行创建，存放本地数据，Git 忽略
    └── semi_supervised_outputs/    # 运行时生成，Git 忽略
```

## 安装与运行

需要 Python 3.10 或更高版本。依赖清单给出允许安装的版本范围，并非所有版本组合都经过测试。

### 1. 安装依赖

下载或克隆仓库后，在项目根目录打开终端。Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m ipykernel install --user --name steel-preprocessing --display-name "Python (steel-preprocessing)"
.\.venv\Scripts\python.exe -m jupyterlab
```

macOS / Linux：

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m ipykernel install --user --name steel-preprocessing --display-name "Python (steel-preprocessing)"
.venv/bin/python -m jupyterlab
```

在 JupyterLab 中打开 `数据预处理/数据预处理.ipynb`，选择 `Python (steel-preprocessing)` 内核。也可在支持 Jupyter 的编辑器中选择项目虚拟环境。

### 2. 准备 CSV

默认将数据放入 `数据预处理/data/input.csv`。以下是字段说明，实际特征名称可不同：

| 字段 | 含义与要求 |
| --- | --- |
| `MILL_OCCR_DATE` | 生产时间，能够被 `pandas.to_datetime` 解析，例如 `2026-03-01 08:00:00` |
| `HEAT_NO` | 炉号；同一炉次使用一致的标识 |
| `RTT_RPDOT2` | 数值回归目标；缺测时留空，不用 0 代替缺失 |
| 其他数值列 | 化学成分、工艺参数等候选输入特征；至少有一列在清洗后可用 |

字段名不能重复。特征中的无穷值需预先处理，数值字符串应转换为数值类型。非数值列及 `EXCLUDE_COLS` 中的列不作为模型特征。应根据实际预测时可获得的信息检查特征清单，排除目标衍生字段及事后检测结果。

完整半监督流程要求训练时段至少有 **5 个炉号**，内部训练清洗和裁剪后至少有两条有效真实标签，且内部验证集有可用真实标签。要获得有效测试指标，测试时段也需有真实标签；R² 至少需要两条测试样本。没有可用无标签样本时可以保留纯监督基线。

### 3. 修改参数并运行

在 Notebook 的“参数与数据”单元格中检查路径和时间切点：

```python
file_path = "data/input.csv"
config = PreprocessConfig(
    target_col="RTT_RPDOT2",
    cutoff_date="2026-03-16",  # 根据自己的数据时间范围调整
    feature_missing_threshold=0.50,
    row_missing_threshold=0.40,
    k=5,
    dense_ratio=0.10,
    dense_keep_ratio=0.50,
    random_state=42,
)
df = pd.read_csv(file_path)
```

相对路径以 **内核当前工作目录** 为准，通常是 Notebook 所在目录；可运行 `Path.cwd()` 检查（先 `from pathlib import Path`）。若文件采用其他编码，需为 `pd.read_csv` 设置对应的 `encoding`。

首次运行请重启内核并按顺序运行全部单元格。最后的导出单元格会覆盖当前工作目录中 `semi_supervised_outputs/` 下同名结果文件。

## 主要参数

| 配置 | 参数 | 默认值 | 含义 |
| --- | --- | --- | --- |
| 预处理 | `cutoff_date` | `2026-03-16` | 训练和测试时间切点 |
| 预处理 | `feature_missing_threshold` | `0.50` | 删除缺失率超过该值的候选特征 |
| 预处理 | `row_missing_threshold` | `0.40` | 隔离所选特征缺失比例超过该值的样本 |
| 预处理 | `k` | `5` | KNN 近邻数，样本不足时自动减少 |
| 预处理 | `dense_ratio` | `0.10` | 根据近邻距离分位数确定密集候选样本 |
| 预处理 | `dense_keep_ratio` | `0.50` | 密集候选样本保留比例 |
| 半监督 | `validation_fraction` | `0.20` | 用于确定内部时间切点的炉号比例参数，非精确行数比例 |
| 半监督 | `max_iter` | `3` | 最多自训练轮数 |
| 半监督 | `batch_fraction` | `0.25` | 每轮加入数量上限为初始无标签池比例向上取整，最少 1 条 |
| 半监督 | `pseudo_weight` | `0.30` | 伪标签训练权重；真实标签权重为 1 |
| 半监督 | `uncertainty_ratio` | `0.35` | 与真实训练标签标准差相乘，得到树间分歧筛选阈值 |
| 半监督 | `n_estimators` | `200` | 随机森林树数量 |

近邻距离并列时，密集候选样本数量可能超过 `dense_ratio` 对应的比例；按保留比例计算的数量四舍五入为 0 时，当前实现保留整个候选组。

## 获取与理解结果

| 对象或文件 | 用途 |
| --- | --- |
| `X_train` / `y_train` | 清洗与 KNN 裁剪后的真实标签训练数据 |
| `X_test` / `y_test` | 有标签测试数据 |
| `X_train_unlabeled` / `X_test_unlabeled` | 各时段可用的无标签特征；测试时段不参与训练 |
| `result.raw` / `result.rejected` / `result.knn_removed` | 原始分组、严重缺失隔离样本、KNN 删除样本 |
| `result.periods.purge` / `result.periods.invalid` | 跨切点炉次、日期或炉号无效的样本 |
| `X_train_semi` / `y_train_semi` / `sample_weight_semi` | 半监督候选增强训练数据及权重 |
| `recommended_model` | 按内部验证结果推荐的模型 |
| `train_augmented.csv` | 真实标签与入选伪标签组成的增强训练集 |
| `pseudo_label_audit.csv` | 全部可用训练无标签样本的预测、分歧与入选状态 |
| `remaining_unlabeled.csv` | 未被采用的可用训练无标签样本 |
| `validation_history.csv` | 内部验证各轮记录 |
| `test_metrics.csv` | 纯监督基线和半监督候选的测试指标 |

增强训练集中的 `_LABEL_SOURCE` 区分 `measured` 与 `pseudo`，`_ORIGINAL_TARGET` 保留原目标，`_SAMPLE_WEIGHT` 保存权重。审计 CSV 的 `train_row_id` 是重建后的训练分区行索引，不等同于原始 CSV 行号或业务主键。

后续建模应使用 `X_train_semi` 或 `result.preprocessor.model_feature_cols` 指定的特征，不能把目标、标签来源、样本权重和伪标签审计字段混入特征。

## 适用范围与限制

- 伪标签是模型估计，不能当成实测数据；树间预测标准差是启发式筛选分数，不是校准后的置信区间。
- 即使内部验证没有改善，仍可能导出半监督候选数据。判断推荐模型应使用 `recommended_model`，不要直接将 `semi_model` 视为最优模型。
- 时间与炉号隔离减少特定泄漏风险，但仍需自行核对特征含义、重复样本和实际预测场景。
- 原始数据未公开，因此不提供可复现的生产数据精度承诺。项目尚未提供模型持久化、服务部署或独立的自动化测试套件。
- Notebook 内置了样本数量守恒、标签状态、训练测试炉号隔离及索引对齐等断言。

当前验证：在本地 Python 3.14.8、NumPy 2.5.3、pandas 3.0.6、scikit-learn 1.9.1 环境中，使用 120 条合成样本完成预处理、伪标签训练和测试评估的冒烟检查（10 棵树、最多 2 轮）。这不代表原始生产数据效果，也未覆盖全新环境安装或 JupyterLab 界面操作。

## 参与贡献

欢迎通过 GitHub Issues 报告问题或提出需求，通过 Pull Request 提交修改。报告问题时请附 Python / 依赖版本、复现步骤和可公开的最小示例；涉及数据时优先使用合成样本。

修改 Notebook 后请重启内核验证执行顺序，并在提交前清除所有输出、保存文件。解释算法改动时，请说明其对时间划分、真实标签和伪标签处理的影响。

## 许可证与发布指南

本项目代码和随附文档采用 [MIT License](LICENSE)，允许使用、修改、分发和商业使用，分发时需保留版权及许可声明。软件按原样提供。原始数据及第三方依赖的权利不因本项目许可证而改变，依赖仍遵守各自许可证。

首次上传、开源文件的作用和后续维护步骤见 [GitHub 开源发布指南](docs/GITHUB_PUBLISHING.md)。
