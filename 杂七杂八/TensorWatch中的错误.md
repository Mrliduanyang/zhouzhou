# TensorWatch中的错误

最近，在基于TensorWatch做一些验证实验。TensorWatch中有一个功能是统计DNN模型在推理过程中的一些性能数据，如每一层的FLOPs、MAdd、推理时间等。但在验证过程中，发现了TensorWatch中关于统计推理时间的一个错误。

## 0.错误来源

原本是想观察在CNN模型中进行分区卷积的效果。分区卷积就是在CNN推理到某一层时，把一个大的输入特征图分成几个小规模的特征图分别进行正向推理，然后再把卷积结果合并。在这个过程中就需要统计每一个小规模特征图的卷积计算时间，但意外发现，TensorWatch中只保留了同一卷积层最后一次调用正向传播的统计信息，这肯定不太对，于是去扒拉扒拉TensorWatch的源代码。

## 1.TensorWatch原理分析

TensorWatch是借助PyTorch中的Hook特性来获取CNN推理的中间结果，并将这些结果整理后格式化输出。TensorWatch中模型分析的最顶层代码如下所示：

```python
# tensorwatch\tensorwatch\model_graph\torchstat\analyzer.py
def analyze(model: nn.Module, img, query_granularity: int):
    # 存储forward前置hooks，forward后置hooks
    pre_hooks, post_hooks = [], []
    # 存储CNN中每个Module的统计结果
    stats: OrderedDict[str, ModuleStats] = OrderedDict()
    try:
        # 为CNN每个Module注册hook，初始化stats
        _for_leaf(model, _register_hooks, pre_hooks, post_hooks, stats)
        # 进行CNN推理
        with torch.no_grad():
            model(return_loss=False, rescale=True, **img)
        # 将每个Module的统计结果组织成树
        stat_tree = _convert_leaf_modules_to_stat_tree(stats)
        # 按照指定的查询粒度从结果树中获取结果
        return stat_tree.get_collected_stat_nodes(query_granularity)
    finally:
		# other code
```

我们再来细看每个Module的前置hook和后置hook的功能。

```python
# tensorwatch\tensorwatch\model_graph\torchstat\analyzer.py
def _forward_pre_hook(module_stats:ModuleStats, module:nn.Module, input):
    assert not module_stats.done
    # 记录Module起始时间
    module_stats.start_time = time.time()
```

_forward_pre_hook是为CNN中每个Module对应的module_stats的start_time属性设置值，以记录该Module的推理起始时间。

```python
# tensorwatch\tensorwatch\model_graph\torchstat\analyzer.py
def _forward_post_hook(module_stats:ModuleStats, module:nn.Module, input, output):
    assert not module_stats.done
	# 记录Module结束时间
    module_stats.end_time = time.time()
    # 计算Module持续时间
    module_stats.duration = module_stats.end_time-module_stats.start_time
	# other code，记录Module中其他统计项数据
```

_forward_post_hook记录该Module的持续时间和其他的统计项数据。

## 2.问题分析

在上面两段代码中，存在着两个严重的问题：

- module_stats.end_time = time.time()，记录结束时间。在PyTorch里面，程序的执行是异步的，如果采用上述方式，得到的时间会很短，因为没有等待GPU完成计算就记录了结束时间。

- module_stats.duration = module_stats.end_time-module_stats.start_time，记录持续时间。如果一个Module正向传播多次，按上述代码，只会记录最后一次正向传播的持续时间，而不会记录每次正向传播的时间。

## 3.问题修复

```python
def _forward_pre_hook(module_stats:ModuleStats, module:nn.Module, input):
    assert not module_stats.done
    # 记录Module起始时间
    torch.cuda.synchronize()
    module_stats.start_time = time.time()

def _forward_post_hook(module_stats:ModuleStats, module:nn.Module, input, output):
    assert not module_stats.done
	# 记录Module结束时间
    torch.cuda.synchronize()
    module_stats.end_time = time.time()
    # 用list存储Module每次正向推理的持续时间
    # module_stats.duration = module_stats.end_time-module_stats.start_time
    module_stats.duration.append(module_stats.end_time - module_stats.start_time)
	# other code，记录Module中其他统计项数据
```

除了如上所示的修改外，还需要修改几个类：

```python
# tensorwatch\tensorwatch\model_graph\torchstat\analyzer.py
class ModuleStats:
    def __init__(self, name) -> None:
        # self.duration = 0.0
        self.duration = []

# tensorwatch\tensorwatch\model_graph\torchstat\stat_tree.py        
class StatNode(object):
    def __init__(self, name=str(), parent=None):
        # duration改为list，解决同一Module调用多次但只保留最后一次的问题
        # self.duration = 0
        self._duration = []
        
    @property
    def duration(self):
        # total_duration = sum(self._duration)
        # for child in self.children:
        #     total_duration += child.duration
        # return total_duration
        return self._duration
```