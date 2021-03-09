# ResNets

# 残差网络

In this chapter, we will build on top of the CNNs introduced in the previous chapter and explain to you the ResNet (residual network) architecture. It was introduced in 2015 by Kaiming He et al. in the article ["Deep Residual Learning for Image Recognition"](https://arxiv.org/abs/1512.03385) and is by far the most used model architecture nowadays. More recent developments in image models almost always use the same trick of residual connections, and most of the time, they are just a tweak of the original ResNet.

这一章节，我们是在上一章节介绍的CNN之上建设的，并向你解释残差网络（ResNet）架构。这个架构在2015年由何凯明等人编写的文章[“面向图像识别的深度残差学习”](https://arxiv.org/abs/1512.03385)中引入的，现如今这是目前为止最常用的模型架构。图像模型最新发展总会使用相同的残差连接技巧，大多数时间他们只会对原始ResNet做微调。

We will first show you the basic ResNet as it was first designed, then explain to you what modern tweaks make it more performant. But first, we will need a problem a little bit more difficult than the MNIST dataset, since we are already close to 100% accuracy with a regular CNN on it.

我会首先给你展示最初设计的基础ResNet，然后向你解释哪些最新调整使得这个架构更加高效。但首先，我们需要一个比MNIST数据集更有难度的问题，因为我们用一个常规CNN在这些基础问题上已经接近了100%的精度。

## Going Back to Imagenette

## 回到Imagenette

It's going to be tough to judge any improvements we make to our models when we are already at an accuracy that is as high as we saw on MNIST in the previous chapter, so we will tackle a tougher image classification problem by going back to Imagenette. We'll stick with small images to keep things reasonably fast.

当我们已经达到上一章节MNIST上所见到的精度高度时，我们都很难判断对模型做出的任何改进，所以我们回到Imagenette处理更难的图像分类问题。我们会放置少量的图像以保证处理相当的快。

Let's grab the data—we'll use the already-resized 160 px version to make things faster still, and will random crop to 128 px:

我们会使用已经调整尺寸为160像素版本的数据，以确保处理更快。我们来抓取数据，并随机剪切为128像素：

```
def get_data(url, presize, resize):
    path = untar_data(url)
    return DataBlock(
        blocks=(ImageBlock, CategoryBlock), get_items=get_image_files, 
        splitter=GrandparentSplitter(valid_name='val'),
        get_y=parent_label, item_tfms=Resize(presize),
        batch_tfms=[*aug_transforms(min_scale=0.5, size=resize),
                    Normalize.from_stats(*imagenet_stats)],
    ).dataloaders(path, bs=128)
```

```
dls = get_data(URLs.IMAGENETTE_160, 160, 128)
```

```
dls.show_batch(max_n=4)
```

Out: <img src="./_v_images/show_batch1.png" style="zoom:100%;" />

When we looked at MNIST we were dealing with 28×28-pixel images. For Imagenette we are going to be training with 128×128-pixel images. Later, we would like to be able to use larger images as well—at least as big as 224×224 pixels, the ImageNet standard. Do you recall how we managed to get a single vector of activations for each image out of the MNIST convolutional neural network?

当我们使用MNIST时，我们处理的是 28×28 像素图像。对于Imagenette我们会用 128×128 像素图像做训练。稍后，我们也想要能够使用更大的图像，至少大小 224×224 像素，这是ImageNet的标准。你还记得我们设法获得MNIST卷积神经网络的每张图像激活的单一向量吗？

The approach we used was to ensure that there were enough stride-2 convolutions such that the final layer would have a grid size of 1. Then we just flattened out the unit axes that we ended up with, to get a vector for each image (so, a matrix of activations for a mini-batch). We could do the same thing for Imagenette, but that would cause two problems:

- We'd need lots of stride-2 layers to make our grid 1×1 at the end—perhaps more than we would otherwise choose.
- The model would not work on images of any size other than the size we originally trained on.

我们所使用的方法确保有足够多的步长2卷积，这样最后层就会有一个格子大小。最终我们只是摊平单元轴，来获得每张图像的向量（这样，对于最小批次的激活矩阵）。我们能够对Imagenette做同样的事情，但是这会产生两个问题：

- 我们需要很多步长2层来使得最后是 1×1 表格。也许比我们原本选择的更多。
- 除了我们最初训练的尺寸，模型也许不会在其它任何大小的图像上运行。

One approach to dealing with the first of these issues would be to flatten the final convolutional layer in a way that handles a grid size other than 1×1. That is, we could simply flatten a matrix into a vector as we have done before, by laying out each row after the previous row. In fact, this is the approach that convolutional neural networks up until 2013 nearly always took. The most famous example is the 2013 ImageNet winner VGG, still sometimes used today. But there was another problem with this architecture: not only did it not work with images other than those of the same size used in the training set, but it required a lot of memory, because flattening out the convolutional layer resulted in many activations being fed into the final layers. Therefore, the weight matrices of the final layers were enormous.

处理第一个问题的方法是摊平最后的卷积层，这样来处理一个大于 1×1 的表格大小。即，我们能够像之前做的那样简单的摊平一个矩阵为一个向量，通过摆放每一行在之前列之后。真实上，这是一个直到2013年几乎总被采用的卷积神经网络。最著名的例子是2013年的ImageNet获胜者VGG，今年有时仍被使用。但是这个架构有另外一个问题：不但它除了训练集中使用相同之外的尺寸图像不能处理，而且它需要大量内存，因此摊平卷积层导致很多激活喂给最后一层。因此最后层的权重矩阵是巨大的。

This problem was solved through the creation of *fully convolutional networks*. The trick in fully convolutional networks is to take the average of activations across a convolutional grid. In other words, we can simply use this function:

这个问题通过创建*全卷积网络*解决了。全卷积网络技巧是通过一个卷积表格秋激活的平均值，我们能够使用下面这个简单函数来实现：

```
def avg_pool(x): return x.mean((2,3))
```

As you see, it is taking the mean over the x- and y-axes. This function will always convert a grid of activations into a single activation per image. PyTorch provides a slightly more versatile module called `nn.AdaptiveAvgPool2d`, which averages a grid of activations into whatever sized destination you require (although we nearly always use a size of 1).

如你所见，它在x和y轴上求平均值。这个函数会把每张图像的激活表格转换为一个单激活。PyTorch提供了多用途模块`nn.AdaptiveAvgPool2d`，它把激活的表格平均为任何我们最终需要的尺寸（虽然我们几乎总是使用尺寸1）。

A fully convolutional network, therefore, has a number of convolutional layers, some of which will be stride 2, at the end of which is an adaptive average pooling layer, a flatten layer to remove the unit axes, and finally a linear layer. Here is our first fully convolutional network:

因此一个全卷积网络有许多卷积层，他们中的一些会是步长2，最后是自适应平均池化层，用以移除单元轴的扁平层，及最终的线性层。下面是我们的第一个全卷积网络：

```
def block(ni, nf): return ConvLayer(ni, nf, stride=2)
def get_model():
    return nn.Sequential(
        block(3, 16),
        block(16, 32),
        block(32, 64),
        block(64, 128),
        block(128, 256),
        nn.AdaptiveAvgPool2d(1),
        Flatten(),
        nn.Linear(256, dls.c))
```

We're going to be replacing the implementation of `block` in the network with other variants in a moment, which is why we're not calling it `conv` any more. We're also saving some time by taking advantage of fastai's `ConvLayer`, which that already provides the functionality of `conv` from the last chapter (plus a lot more!).

稍后我们会用其它变体来替换网络中的`block`实现，这是为什么我们没有再称它`conv`的原因。通过求fastai的`ConvLayer`平均值我们也节省了一些时间，在上一章它已经提供了实用的`conv`功能（还有更多！）

> stop: Consider this question: would this approach makes sense for an optical character recognition (OCR) problem such as MNIST? The vast majority of practitioners tackling OCR and similar problems tend to use fully convolutional networks, because that's what nearly everybody learns nowadays. But it really doesn't make any sense! You can't decide, for instance, whether a number is a 3 or an 8 by slicing it into small pieces, jumbling them up, and deciding whether on average each piece looks like a 3 or an 8. But that's what adaptive average pooling effectively does! Fully convolutional networks are only really a good choice for objects that don't have a single correct orientation or size (e.g., like most natural photos).

> 暂停：思考这个问题：这个方法对于如MNIST这种光学字符识别（OCR）问题有意义吗？广泛的行业人员与OCR和类似问题打交道倾向使用全神经网络，因为现如今这是几乎每个人都学过的。但是它真的没有任何意义！例如，你不能通过把他们切成很小的小块来决定一个数字是3还是8，弄碎它们并判断每块上的平均看起来像3或是8。但这是自适应平均池化实际做的事情！对于没有单一正确方向或尺寸（例如，像大多数自然照片）目标，全卷积网络确实是一个好的选择。

Once we are done with our convolutional layers, we will get activations of size `bs x ch x h x w` (batch size, a certain number of channels, height, and width). We want to convert this to a tensor of size `bs x ch`, so we take the average over the last two dimensions and flatten the trailing 1×1 dimension like we did in our previous model.

一旦我们用卷积层做完，我们会得到尺寸为 `bs x ch x h x w`的激活（批次尺寸，确定的通道数，高和宽）。我们希望把这个激活形状转换为尺寸为`bs x ch`的张量，所以我们在最后两个轴上求平均值，并像我们之前章节做的那样扁平化尾部的 1×1 维。

This is different from regular pooling in the sense that those layers will generally take the average (for average pooling) or the maximum (for max pooling) of a window of a given size. For instance, max pooling layers of size 2, which were very popular in older CNNs, reduce the size of our image by half on each dimension by taking the maximum of each 2×2 window (with a stride of 2).

某种意义上这与常规池化是不同的，这些层通常会求一个给定尺寸窗口的平均（平均池化）或最大值（最大池化）。例如，尺寸2的最大池化在老的CNN中非常流行，通过求每个 2×2 窗口（步长2）的最大值，通过在每个维度上减半压缩了我们图像的尺寸。

As before, we can define a `Learner` with our custom model and then train it on the data we grabbed earlier:

以前，我们能够用自定义模型定义`Learner`，多面手在我们之前抓取的数据上训练它：

```
def get_learner(m):
    return Learner(dls, m, loss_func=nn.CrossEntropyLoss(), metrics=accuracy
                  ).to_fp16()

learn = get_learner(get_model())
```

```
learn.lr_find()
```

Out: (0.47863011360168456, 3.981071710586548)

 ![img](data:image/png;base64,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)

3e-3 is often a good learning rate for CNNs, and that appears to be the case here too, so let's try that:

对于CNN 3e-3通常是一个合适的学习率，在这里的例子中好像也是这样，让我们尝试一下：

```
learn.fit_one_cycle(5, 3e-3)
```

| epoch | train_loss | valid_loss | accuracy |  time |
| ----: | ---------: | ---------: | -------: | ----: |
|     0 |   1.901582 |   2.155090 | 0.325350 | 00:07 |
|     1 |   1.559855 |   1.586795 | 0.507771 | 00:07 |
|     2 |   1.296350 |   1.295499 | 0.571720 | 00:07 |
|     3 |   1.144139 |   1.139257 | 0.639236 | 00:07 |
|     4 |   1.049770 |   1.092619 | 0.659108 | 00:07 |

That's a pretty good start, considering we have to pick the correct one of 10 categories, and we're training from scratch for just 5 epochs! We can do way better than this using a deeper mode, but just stacking new layers won't really improve our results (you can try and see for yourself!). To work around this problem, ResNets introduce the idea of *skip connections*. We'll explore those and other aspects of ResNets in the next section.

考虑到我们必须从10个分类中选择一个正确的，我们从零开始训练只有5个周期，这是一个非常好的开始！使用一个更深的模型我们能够做的比这个结果更好，但只是堆砌新的层不会真正改善我们的结果（你可以尝试一下并自己看一下！）。围绕这个问题的处理，残差网络 引入了*跳跃连接*思想。下一部分我们会探索这个方法和残差网络的其它部分。