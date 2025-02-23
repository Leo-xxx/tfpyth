## 一行代码切换TensorFlow与PyTorch，模型训练也能用俩框架

[机器之心](javascript:void(0);) *昨天*

机器之心报道

**参与：****思源**

> 你是否有时要用 PyTorch，有时又要跑 TensorFlow？这个项目就是你需要的，你可以在训练中同时使用两个框架，并端到端地转换模型。也就是说 TensorFlow 写的计算图可以作为某个函数，直接应用到 Torch 的张量上，这操作也是很厉害了。



在早两天开源的 TfPyTh 中，不论是 TensorFlow 还是 PyTorch 计算图，它们都可以包装成一个可微函数，并在另一个框架中高效完成前向与[反向传播](https://mp.weixin.qq.com/s?__biz=MzA3MzI4MjgzMw==&mid=2650765856&idx=3&sn=26a9de9b7f4152ff14d2f6ee478848ba&chksm=871abe5eb06d374859b744b79310a0c9dde8d11d898b3e8736a9436a026e0de6d724b7756ee9&scene=0&xtrack=1&key=3e64675c4af2e8c19adbbd8c106fe769d0ac8c942cb960591b144e67946cab735c830af1bac3907540e1b73c6355824f96c3ea5c986e57d52fdfc71d73f59e40893b64b43d4e0b66e3cc9d603f8b5a8d&ascene=1&uin=MjMzNDA2ODYyNQ%3D%3D&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=iqn5fxyAYAEcbOWN8K0hTmIdnQAEbGoAMytUHUJn7mS3BliHEI0JRQI4B417Pox7)。很显然，这样的框架交互，能节省很多重写代码的麻烦事。

- 项目地址：https://github.com/BlackHC/TfPyTh

**为什么框架间的交互很重要**

目前 GitHub 上有很多优质的开源模型，它们大部分都是用 PyTorch 和 TensorFlow 写的。如果我们想要在自己的项目中调用某个开源模型，那么它们最好都使用相同的框架，不同框架间的对接会带来各种问题。当然要是不怕麻烦，也可以用不同的框架重写一遍。

![img](https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gW9g6EYq9Xa6pGIgOORoSGUCuicJRicJCyMhxFtFtRQr3WEbyEkqU2LDLGPyDRXZJwcVzlESetaGaJog/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

以前 TensorFlow 和 PyTorch 经常会用来对比，讨论哪个才是更好的深度学习框架。但是它们之间就不能友好相处么，模型在两者之间的相互迁移应该能带来更多的便利。

在此之前，Facebook 和微软就尝试过另一种方式，即神经网络交换格式 ONNX。直观而言，该工具定义了一种通用的计算图，不同深度学习框架构建的计算图都能转化为它。虽然目前 ONNX 已经原生支持 MXNet、PyTorch 和 Caffe2 等大多数框架，但是像 TensorFlow 或 Keras 之类的只能通过第三方转换器转换为 ONNX 格式。

而且比较重要的一点是，现阶段 ONNX 只支持推理，导入的模型都需要在原框架完成训练。所以，想要加入其它框架的模型，还是得手动转写成相同框架，再执行训练。

**神奇的转换库 TfPyTh**

既然 ONNX 无法解决训练问题，那么就轮到 TfPyTh 这类项目出场了，它无需改写已有的代码就能在框架间自由转换。具体而言，TfPyTh 允许我们将 TensorFlow 计算图包装成一个可调用、[可微分](https://mp.weixin.qq.com/s?__biz=MzA3MzI4MjgzMw==&mid=2650765856&idx=3&sn=26a9de9b7f4152ff14d2f6ee478848ba&chksm=871abe5eb06d374859b744b79310a0c9dde8d11d898b3e8736a9436a026e0de6d724b7756ee9&scene=0&xtrack=1&key=3e64675c4af2e8c19adbbd8c106fe769d0ac8c942cb960591b144e67946cab735c830af1bac3907540e1b73c6355824f96c3ea5c986e57d52fdfc71d73f59e40893b64b43d4e0b66e3cc9d603f8b5a8d&ascene=1&uin=MjMzNDA2ODYyNQ%3D%3D&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=iqn5fxyAYAEcbOWN8K0hTmIdnQAEbGoAMytUHUJn7mS3BliHEI0JRQI4B417Pox7)的简单函数，然后 PyTorch 就能直接调用它完成计算。反过来也是同样的，TensorFlow 也能直接调用转换后的 PyTorch 计算图。

因为转换后的模块是可微的，那么正向和反向传播都没什么问题。不过项目作者也表示该项目还不太完美，开源 3 天以来会有一些小的问题。例如张量必须通过 CPU 进行复制与路由，直到 TensorFlow 支持__cuda_array_interface 相关功能才能解决。

目前 TfPyTh 主要支持三大方法：

- torch_from_tensorflow：创建一个 PyTorch 可微函数，并给定 TensorFlow 占位符输入计算张量输出；
- eager_tensorflow_from_torch：从 PyTorch 创建一个 Eager TensorFlow 函数；
- tensorflow_from_torch：从 PyTorch 创建一个 TensorFlow 运算子或张量。

**TfPyTh 示例**

如下所示为 torch_from_tensorflow 的使用案例，我们会用 TensorFlow 创建一个简单的静态计算图，然后传入 PyTorch 张量进行计算。

```
import tensorflow as tf
import torch as th
import numpy as np
import tfpyth

session = tf.Session()
def get_torch_function():
    a = tf.placeholder(tf.float32, name='a')
    b = tf.placeholder(tf.float32, name='b')
    c = 3 * a + 4 * b * b

    f = tfpyth.torch_from_tensorflow(session, [a, b], c).apply
    return f

f = get_torch_function()
a = th.tensor(1, dtype=th.float32, requires_grad=True)
b = th.tensor(3, dtype=th.float32, requires_grad=True)
x = f(a, b)
assert x == 39.

x.backward()
assert np.allclose((a.grad, b.grad), (3., 24.))
```



我们可以发现，基本上 TensorFlow 完成的就是一般的运算，例如设置占位符和建立计算流程等。TF 的静态计算图可以通过 session 传递到 TfPyTh 库中，然后就产生了一个新的可微函数。后面我们可以将该函数用于模型的某个计算部分，再进行训练也就没什么问题了。*![img](https://mmbiz.qpic.cn/mmbiz_png/KmXPKA19gW8Zfpicd40EribGuaFicDBCRH6IOu1Rnc4T3W3J1wE0j6kQ6GorRSgicib0fmNrj3yzlokup2jia9Z0YVeA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)*



**本文为机器之心报道，转载请联系本公众号获得授权。**

✄------------------------------------------------

**加入机器之心（全职记者 / 实习生）：hr@jiqizhixin.com**

**投稿或寻求报道：content@jiqizhixin.com**

**广告 & 商务合作：bd@jiqizhixin.com**









微信扫一扫
关注该公众号