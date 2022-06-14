# task01-熟悉torch-rechub框架设计与使用方法

## 1 Torch-rechub框架

### 1.1 框架优势/特性

- `scikit-learn`风格易用的API（`fit`、`predict`），开箱即用
- 模型训练与模型定义解耦，易拓展，可针对不同类型的模型设置不同的训练机制
- 支持`pandas`的`DataFrame`、`Dict`等数据类型的输入，降低上手成本
- 高度模块化，支持常见 Layer，容易调用组装形成新的模型
  - LR、MLP、FM、FFM、CIN
  - target-attention、self-attention、transformer
- 支持常见排序模型
  - WideDeep、DeepFM、DIN、DCN、xDeepFM等
- 支持常见召回模型
  - DSSM、YoutubeDNN、YoutubeDSSM、FacebookEBR、MIND等
- 丰富的多任务学习支持
  - SharedBottom、ESMM、MMOE、PLE、AITM等模型
  - GradNorm、UWL、MetaBanlance等动态loss加权机制
- 聚焦更生态化的推荐场景
- 支持丰富的训练机制

### 1.2 整体架构

![](material/ch1-1.png)

- Data类
  - Dense Feature: 数值型特征
  - Sparse Feature：类别型特征（通过labelEncoder得到embedding）
  - Sequence Feature：序列特征

- Model类

  1. 通用layer
     - 浅层特征
       - LR, MLP, EmbeddinLayer
     - 深层特征
       - FM、FFM、CIN
       - self-attention、target-attention、transformer

  2. 总体模型

     - 排序模型：WideDeep、DeepFM、DCN、xDeepFM、DIN、DIEN、SIM

     - 召回模型：DSSM、YoutubeDNN、YoutubeSBC、FaceBookDSSM、Gru4Rec、MIND、SASRec、ComiRec
     - 多任务模型：SharedBottom、ESMM、MMOE、PLE、AITM

- Trainer类

  目前支持三种Trainer

  - CTRTrainer: 精排模型训练与评估，支持BCE Loss
  - MTLTrainer: 多任务排序模型训练与评估，分类任务支持BCE Loss，回归任务支持MSE Loss
  - MatchTrainer：召回模型训练与评估，
    Point-wise样本构造：BCE Loss
    Pair-wise样本构造：BPR Hinge Loss
    List-wise样本构造：softmax Loss
    向量化召回：使用annoy

## 2. 构建案例

完整代码见：https://colab.research.google.com/drive/1SWUj0DTvQ2hixDONc1REsHSYu4hUJMvg?usp=sharing

这里介绍一下模型搭建的部分，略过数据处理

```python
# Data_loader
# x: the training feature
# y: labels
from torch_rechub.basic.features import DenseFeature, SparseFeature

dense_feas = [DenseFeature(feature_name) for feature_name in dense_features]
sparse_feas = [SparseFeature(feature_name, vocab_size=data[feature_name].nunique(), embed_dim=16) for feature_name in sparse_features]
y = data["label"]
del data["label"]
x = data

data_generator = DataGenerator(x, y)
```

```python
# train_val split
batch_size = 2048
# split 70% data for training, 10% for validating and the remaining 20% for testing
train_ratio = 0.7
val_ratio = 0.1
train_dataloader, val_dataloader, test_dataloader = data_generator.generate_dataloader(split_ratio=[train_ratio, val_ratio], batch_size=batch_size)
```

```python
# parameter
learning_rate = 1e-3
weight_decay = 1e-3
epoch = 2
mlp_params={
    "dims": [256, 128], 
    "dropout": 0.2, 
    "activation": "relu"}
device = 'cuda:0'
save_dir = './models/'

```

```python
# define model(WideDeep as sample)
model = WideDeep(wide_features=dense_feas, deep_features=sparse_feas, mlp_params=mlp_params)

# define optimizer
optimizer_params={
    "lr": learning_rate, 
    "weight_decay": weight_decay}

# define trainer
ctr_trainer = CTRTrainer(model, optimizer_params=optimizer_params, n_epoch=epoch, earlystop_patience=10, 
                         device=device, model_path=save_dir)
```

```python
# training
ctr_trainer.fit(train_dataloader, val_dataloader)
```

```python
# validation
auc = ctr_trainer.evaluate(ctr_trainer.model, test_dataloader)
print(f'test auc: {auc}')
```

