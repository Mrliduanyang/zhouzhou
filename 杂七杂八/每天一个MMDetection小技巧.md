# 每天一个MMDetection小技巧

## 0.MMDetection简介

**MMDetection**是商汤科技开源的基于PyTorch实现的目标检测工具箱。其最大的特点是把目标检测框架分解为组件，并可以通过组合模块的方式来轻松构建一个定制的目标检测框架。把研究人员从之前的寻找模型源代码——阅读源代码——修改源代码——训练模型的复杂模式中解放出来，从此定制目标检测模型成了搭积木。**MMDetection**支持**ResNet**、**VGG**在内的7种骨干网，支持**RPN**，**Faster R-CNN**、**SSD**、**YOLO**等40多种目标检测方法。

## 1.TensorWatch简介

**TensorWatch**是一个调试和可视化工具，可以显示机器学习训练时的实时可视化效果，并能对模型和数据执行一些其他的分析任务。**TensorWatch**中包含一个很实用的模型分析功能——查看模型不同层的统计信息，如：**FLOPs**，**MAdd**，**Params**等。

## 2.MMDetection + TensorWatch

如果把**MMDetection**中的模型和**TensorWatch**中的模型分析相结合，可以给目标检测模型性能研究提供很多基础数据。但是**TensorWatch**是没法直接和**MMDetection**结合的，因为**MMDetection**模型的正向传播是基于**PyTorch**定制的，检测器基类**mmdet/models/detectors/base.py**中的**forward**如下所示：

```python
@auto_fp16(apply_to=('img', ))
def forward(self, img, img_metas, return_loss=True, **kwargs):
    # some other codes
```

在**forward**之前有一个**auto_fp16()**装饰器负责处理**img**，并且我们注意到**forward**的参数列表中除了**img**外，还有表示**img**元数据的**img_metas**参数及其他必要参数。但在**TensorWatch**的模型分析入口处**tensorwatch/model_graph/torchstat/analyzer.py**，对待分析模型的调用如下所示：

```python
def analyze(model:nn.Module, input_size, query_granularity:int):
    assert isinstance(model, nn.Module)
    assert isinstance(input_size, (list, tuple))
    x = torch.rand(*input_size)
    model.eval()
    model(x)
    # some other codes
```

**analyze**接收三个参数，待分析模型**model**，模型输入尺寸**input_size**，分析粒度**query_granularity**，其中**model**是**PyTorch**里**nn.Module**类型，**input_size**是列表或者元组类型，其长度视**model**接收的输入而定。可以看到，**analyze**在执行时，先生成一个**input_size**规模的张量，然后直接传入**model**，这也就限制了在使用**TensorWatch**分析模型时，模型只能是接收一个张量参数的**PyTorch**中的**nn.Module**类型模型。不能在模型中添加其他参数或者为模型添加装饰器，这使得我们无法直接在**TensorWatch**中使用**MMDetection**中的众多模型。

## 3.TensorWatch兼容性改造

```python
def analyze(model: nn.Module, img, query_granularity: int):
    assert isinstance(model, nn.Module)
    # assert isinstance(input_size, (list, tuple))
    with torch.no_grad():
        model(return_loss=False, rescale=True, **img)
    # some other codes
```

我们把原**analyze**中需要根据输入尺寸来生成的张量改为接收外部输入**img**，并取消**input_size**参数及相应的断言。这样以来，在**TensorWatch**中对**model**的调用即和**MMDetection**保持一致。又因为**TensorWatch**是借助**PyTorch**的**Hook**机制来实现对模型运行过程中各层数据的采集，因此上边的改造不会影响**TensorWatch**的分析结果。