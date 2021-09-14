# 增加一个新的分类器

本节相关代码：
```
core/model/abstract_model.py
core/model/meta/*
core/model/metric/*
core/model/pretrain/*
```

我们需要从论文中分类的三种方法，即matric based，meta learning，以及fine tuning，从每种方法中选出一个代表性的方法，描述如何添加这一类别的新的方法。

不过在此之前，需要先了解一下所有分类方法共同的父类`abstract_model`。

```python
class AbstractModel(nn.Module):
    def __init__(self,...)
    	# base info

    @abstractmethod
    def set_forward(self,):
        # inference phase
        pass

    @abstractmethod
    def set_forward_loss(self,):
        # training phase
        pass

    def forward(self, x):
        out = self.emb_func(x)
        return out

    def train(self,):
        # override super's function

    def eval(self,):
        # override super's function

    def _init_network(self,):
        # init all layers

    def _generate_local_targets(self,):
        # formate the few shot labels

    def split_by_episode(self,):
        # split batch by way, shot and query

    def reset_base_info(self,):
        # change way, shot and query
```

+ `__init__`：初始化函数，用于初始化一些小样本学习中常用的如way，shot，query这样的参数设置。
+ `set_forward`：用于推理阶段调用，返回分类输出以及准确率。
+ `set_forward_loss`：用于训练阶段调用，返回分类输出、准确率以及前向损失。
+ `forward`：重写`pytorch`的`Module`中的`forward`函数，返回`backbone`的输出。
+ `train`：重写`pytorch`的`Module`中的`train`函数，用于解除`BatchNorm`层的参数固定。
+ `eval`：重写`pytorch`的`Module`中的`eval`函数，用于固定`BatchNorm`层的参数。
+ `_init_network`：用于初始化所有网络。
+ `_generate_local_targets`：用于生成小样本学习的任务中所使用的`target`。
+ `split_by_episode`：将输入按照`episode_size,way,shot,query`切分好便于后续处理。提供了几种切分方式。
+ `reset_base_info`：改变小样本学习的`way,shot,query`等设置。

其中，添加新的方法必须要重写`set_forward`以及`set_forward_loss`这两个函数，其他的函数都可以根据所实现方法的需要来调用。

注意，为了新添加的方法能够通过反射机制调用到，需要在对应方法类型的目录下的`__init__.py`文件中加上一行：

```python
from NewMethodFileName import *
```

## metric based

接下来将以`DN4`为例，描述如何在`LibFewShot`中添加一个新的`metric based classifier`。

`metric based`方法有一个共同的父类`MetricModel`，继承了`AbstractModel`。

```python
class MetricModel(AbstractModel):
    def __init__(self,):
        ...

    @abstractmethod
    def set_forward(self, *args, **kwargs):
        pass

    @abstractmethod
    def set_forward_loss(self, *args, **kwargs):
        pass

    def forward(self, x):
        out = self.emb_func(x)
        return out
```

由于`metric based`方法的`pipeline`的方法大多比较简单，因此只是继承了`abstract_model`，并没有做其他修改。

在`core/model/metric/`下添加`dn4.py`文件。首先创建`DN4`的模型类：

```python
class DN4(MetricModel):
    def __init__(self, way_num, shot_num, query_num, emb_func, device, n_k=3):
        # base info
        super(DN4Layer, self).__init__()
        self.way_num = way_num
        self.shot_num = shot_num
        self.query_num = query_num
        self.n_k = n_k
        self.loss_func = nn.CrossEntropyLoss()

    def set_forward(self, batch):
        # inference phase
        """
        :param batch: (images, labels)
        :param batch.images: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query),C,H,W]
        :param batch.labels: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query), ]
        :return: net output and accuracy
        """
        image, global_target = batch
        image = image.to(self.device)
        episode_size = image.size(0) // (
            self.way_num * (self.shot_num + self.query_num)
        )
        feat = self.emb_func(image)
        support_feat, query_feat, support_target, query_target = self.split_by_episode(
            feat, mode=2
        )

        t, wq, c, h, w = query_feat.size()
        _, ws, _, _, _ = support_feat.size()

        # t, wq, c, hw -> t, wq, hw, c -> t, wq, 1, hw, c
        query_feat = query_feat.view(
            t, self.way_num * self.query_num, c, h * w
        ).permute(0, 1, 3, 2)
        query_feat = F.normalize(query_feat, p=2, dim=2).unsqueeze(2)

        # t, ws, c, h, w -> t, w, s, c, hw -> t, 1, w, c, shw
        support_feat = (
            support_feat.view(t, self.way_num, self.shot_num, c, h * w)
            .permute(0, 1, 3, 2, 4)
            .contiguous()
            .view(t, self.way_num, c, self.shot_num * h * w)
        )
        support_feat = F.normalize(support_feat, p=2, dim=2).unsqueeze(1)

        # t, wq, w, hw, shw -> t, wq, w, hw, n_k -> t, wq, w
        relation = torch.matmul(query_feat, support_feat)
        topk_value, _ = torch.topk(relation, self.n_k, dim=-1)
        score = torch.sum(topk_value, dim=[3, 4])

        output = score.view(episode_size * self.way_num * self.query_num, self.way_num)
        acc = accuracy(output, query_target)

        return output, acc

    def set_forward_loss(self, batch):
        # training phase
        """
        :param batch: (images, labels)
        :param batch.images: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query),C,H,W]
        :param batch.labels: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query), ]
        :return: net output, accuracy and train loss
        """
        image, global_target = batch
        image = image.to(self.device)
        episode_size = image.size(0) // (
            self.way_num * (self.shot_num + self.query_num)
        )
        emb = self.emb_func(image)
        support_feat, query_feat, support_target, query_target = self.split_by_episode(
            emb, mode=2
        )

        t, wq, c, h, w = query_feat.size()
        _, ws, _, _, _ = support_feat.size()

        # t, wq, c, hw -> t, wq, hw, c -> t, wq, 1, hw, c
        query_feat = query_feat.view(
            t, self.way_num * self.query_num, c, h * w
        ).permute(0, 1, 3, 2)
        query_feat = F.normalize(query_feat, p=2, dim=2).unsqueeze(2)

        # t, ws, c, h, w -> t, w, s, c, hw -> t, 1, w, c, shw
        support_feat = (
            support_feat.view(t, self.way_num, self.shot_num, c, h * w)
            .permute(0, 1, 3, 2, 4)
            .contiguous()
            .view(t, self.way_num, c, self.shot_num * h * w)
        )
        support_feat = F.normalize(support_feat, p=2, dim=2).unsqueeze(1)

        # t, wq, w, hw, shw -> t, wq, w, hw, n_k -> t, wq, w
        relation = torch.matmul(query_feat, support_feat)
        topk_value, _ = torch.topk(relation, self.n_k, dim=-1)
        score = torch.sum(topk_value, dim=[3, 4])

        output = score.view(episode_size * self.way_num * self.query_num, self.way_num)
        loss = self.loss_func(output, query_target)
        acc = accuracy(output, query_target)

        return output, acc, loss
```

在`__init__`中，对分类器可能用到的小样本学习的基本设置进行了初始化，还传入了DN4方法的一个超参数`n_k`。

在`set_forward`与`set_forward_loss`中，需要注意的是`19-27,65-73`行，这部分代码对输入的batch进行处理，提取特征，最后切分为小样本学习中需要使用的`support set`和`query set`的特征。具体来说，为了最大化利用计算资源，我们将所有图像同时经过`backbone`，之后对特征向量进行`support set, query set`的切分。`29-50,75-96`行为DN4方法的计算过程。最终`set_forward`的输出为$output.shape:[episode\_size*way*query,way]，acc:float$，`set_forward_loss`的输出为$output.shape:[episode\_size*way*query,way], acc:float, loss:tensor$。其中`output`需要用户根据方法进行生成，`acc`可以调用`LibFewShot`提供的`accuracy`函数，输入`output, target`就可以得到分类准确率。而`loss`可以使用用户在方法开始时初始化的损失函数，在`set_forward_loss`中使用来得到分类损失。

metric方法中只需要根据自己设计的方法，将输入处理为对应的形式就可以开始训练了。

## meta learning

接下来将以`MAML`为例，描述如何在`LibFewShot`中添加一个新的`meta learning classifier`。

`meta learning`方法有一个共同的父类`MetaModel`，继承了`AbstractModel`。

## fine tuning

接下来将以`Baseline`为例，描述如何在`LibFewShot`中添加一个新的`fine-tuning classifier`。

`fine-tuning`方法有一个共同的父类`PretrainModel`，继承了`AbstractModel`。

```python
class PretrainModel(AbstractModel):
    def __init__(self, way_num, shot_num, query_num, emb_func, device, init_type="normal"):
        super(PretrainModel, self).__init__(way_num, shot_num, query_num, emb_func, device, init_type,
                                            ModelType.FINETUNING)

    @abstractmethod
    def set_forward(self, *args, **kwargs):
        pass

    @abstractmethod
    def set_forward_loss(self, *args, **kwargs):
        pass

    def forward(self, x):
        out = self.emb_func(x)
        return out

    @abstractmethod
    def test_loop(self, *args, **kwargs):
        pass

    @abstractmethod
    def set_forward_adaptation(self, *args, **kwargs):
        pass

    def sub_optimizer(self, model, config):
        kwargs = dict()
        if config["kwargs"] is not None:
            kwargs.update(config["kwargs"])
        return getattr(torch.optim, config["name"])(model.parameters(), **kwargs)
```

`fine-tuning`方法训练时的目标是训练出一个好的特征抽取器，在测试时使用小样本学习的设置，通过`support set`来对模型进行微调。也有的方法是在训练完毕特征抽取器后，再使用小样本学习的训练设置来进行整个模型的微调。为了与`meta learning`的方法统一，我们添加了一个`set_forward_adaptation`抽象函数，用于处理在测试时的前向过程。另外，由于有一些`fine-tuning`方法的测试过程中，也需要训练分类器，因此，添加了一个`sub_optimizer`方法，传入需要优化的参数以及优化的配置参数，返回优化器，用以方便调用。

在`core/model/pretrain/`下添加`baseline.py`文件。首先创建`Baseline`的模型类：

```python
class Baseline(PretrainModel):
    def __init__(self, way_num, shot_num, query_num, emb_func, device, feat_dim, num_class, inner_optim=None, inner_batch_size=4, inner_train_iter=100):
        super(Baseline, self).__init__(way_num, shot_num, query_num, emb_func, device)
        self.feat_dim = feat_dim
        self.num_class = num_class
        self.inner_optim = inner_optim
        self.inner_batch_size = inner_batch_size
        self.inner_train_iter = inner_train_iter

        self.classifier = nn.Linear(self.feat_dim, self.num_class)
        self.loss_func = nn.CrossEntropyLoss()

    def set_forward(self, batch):
        # inference phase
        """
        :param batch: (images, labels)
        :param batch.images: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query),C,H,W]
        :param batch.labels: shape: [episodeSize*way*(shot*augment_times+query*augment_times_query), ]
        :return: net output and accuracy
        """
        image, global_target = batch
        image = image.to(self.device)
        with torch.no_grad():
            feat = self.emb_func(image)
        support_feat, query_feat, support_target, query_target = self.split_by_episode(
            feat, mode=4
        )

        classifier = self.set_forward_adaptation(support_feat, support_target)

        output = classifier(query_feat)
        acc = accuracy(output, query_target)

        return output, acc

    def set_forward_loss(self, batch):
        # training phase
        """
        :param batch: (images, labels)
        :param batch.images: shape: [batch_size*augment_times,C,H,W]
        :param batch.labels: shape: [batch_size*augment_times, ]
        :return: net output, accuracy and train loss
        """
        image, target = batch
        image = image.to(self.device)
        target = target.to(self.device)

        feat = self.emb_func(image)
        output = self.classifier(feat)
        loss = self.loss_func(output, target)
        acc = accuracy(output, target)
        return output, acc, loss

    def set_forward_adaptation(self, support_feat, support_target):
        classifier = nn.Linear(self.feat_dim, self.way_num)
        optimizer = self.sub_optimizer(classifier, self.inner_optim)

        classifier = classifier.to(self.device)

        classifier.train()
        support_size = support_feat.size(0)
        for epoch in range(self.inner_train_iter):
            rand_id = torch.randperm(support_size)
            for i in range(0, support_size, self.inner_batch_size):
                select_id = rand_id[i : min(i + self.inner_batch_size, support_size)]
                batch = support_feat[select_id]
                target = support_target[select_id]

                output = classifier(batch)

                loss = self.loss_func(output, target)

                optimizer.zero_grad()
                loss.backward()
                optimizer.step()

        return classifier
```

`48,49`行为训练过程的主要代码，这段代码就是经典分类方法的训练过程。`50-52`行与经典有监督学习方法一样生成训练过程的输出、准确率、训练损失。而测试阶段与`metric based, meta learning`方法类似都是通过小样本的前向传播过程得到测试阶段的输出以及准确率。`set_forward_adaptation`函数中的内容，是测试阶段的主要过程。其主要是使用backbone提取出的`support set, query set`的特征，使用`support set`的特征来训练一个分类器，并使用`query set`的特征使用分类器进行分类。