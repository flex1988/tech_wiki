# sklearn--Scikit-learn

Scikit-learn（通常简写为 sklearn）是 Python 编程语言中一个自由且开源的机器学习库。

Scikit-learn 并非深度学习库（如 TensorFlow 或 PyTorch），它专注于经典机器学习算法和模型构建的整个工作流。

它的设计核心在于提供简单、高效、一致的 API，帮助用户快速构建、训练和评估模型。

Scikit-learn 提供了几乎所有主流的经典机器学习算法，主要分为以下几大类：

| 模块类别 (Module)                 | 目的 (Goal)         | 包含的常见算法/工具 (Examples)                                                       |
| ----------------------------- | ----------------- | --------------------------------------------------------------------------- |
| 分类 (Classification)           | 预测离散的类别标签。        | 逻辑回归 (Logistic Regression)、支持向量机 (SVM)、K近邻 (K-NN)、决策树、随机森林 (Random Forest)。 |
| 回归 (Regression)               | 预测连续的数值结果。        | 线性回归 (Linear Regression)、岭回归 (Ridge)、套索回归 (Lasso)、弹性网络 (Elastic Net)。       |
| 聚类 (Clustering)               | 在无监督学习中，自动将数据点分组。 | K均值 (K-Means)、DBSCAN、层次聚类 (Agglomerative Clustering)。                       |
| 降维 (Dimensionality Reduction) | 减少特征数量以简化模型和可视化。  | 主成分分析 (PCA)、线性判别分析 (LDA)。                                                   |
| 预处理 (Preprocessing)           | 数据清洗、转换和标准化。      | 特征缩放 (Scaling)、归一化 (Normalization)、独热编码 (One-Hot Encoding)。                 |
| 模型选择 (Model Selection)        | 评估、调整和优化模型。       | 交叉验证 (Cross-Validation)、网格搜索 (Grid Search)、学习曲线。                            |

#### 与其他库的关系

Scikit-learn 的强大在于它与 Python 生态系统的紧密集成：

* NumPy： 作为底层的数据结构库，所有输入数据都需要是 NumPy 数组或可以转换为 NumPy 数组（如 Pandas DataFrame）。
* SciPy： 依赖 SciPy 提供的科学计算功能。
* Pandas： 经常与 Pandas 一起使用，Pandas 负责数据导入和处理，然后将 DataFrame 传递给 Scikit-learn 进行建模。
* Matplotlib/Plotly： 用于模型的评估结果和数据可视化。
