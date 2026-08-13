# LLM 篇

## Attention面经篇

Transformer中的注意力机制(Attention Mechanism)是其核心组成部分之一，主要用于捕捉输入序列中不同位置之间的依赖关系。通过计算每个输入的隐藏状态(hiddenstate)，同时利用注意力机制来解决上下文关联问题。

### 1.Transformer架构

首先介绍 Transformer 的整体结构，下图是 Transformer 用于中英文翻译的整体结构：
![img](LLM 篇.assets/v2-4544255f3f24b7af1e520684ae38403f_1440w.jpg)

可以看到 **Transformer 由 Encoder 和 Decoder 两个部分组成**，Encoder 和 Decoder 都包含 6 个 block。

#### 1.1Transformer 的工作流程 

**第一步：**获取输入句子的每一个单词的表示向量 **X**，**X**由单词的 [Embedding](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=Embedding&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODQ5MDgwNTQsInEiOiJFbWJlZGRpbmciLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoxNjM0MjI5NzksImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.axHbU5OsyXcmIhS90cTOkEcAZV_XTLJgGqmXEsewu6g&zhida_source=entity)（Embedding就是从原始数据提取出来的Feature） 和单词位置的 Embedding 相加得到。

![img](LLM 篇.assets/v2-7dd39c44b0ae45d31a3ae7f39d3f883f_1440w.jpg)

​																													*Transformer 的输入表示*

**第二步：**将得到的单词表示向量矩阵 (如上图所示，每一行是一个单词的表示 **x**) 传入 Encoder 中，经过 6 个 Encoder block 后可以得到句子所有单词的编码信息矩阵 **C**，如下图。单词向量矩阵用 表示， n 是句子中单词个数，d 是表示向量的维度 (论文中 d=512)。每一个 Encoder block 输出的[矩阵维度](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=矩阵维度&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODQ5MDgwNTQsInEiOiLnn6npmLXnu7TluqYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoxNjM0MjI5NzksImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.c1feBp9_h9EcFh_3VTLR6Uy2mbcsbnktQA31EnctQz8&zhida_source=entity)与输入完全一致。

![img](LLM 篇.assets/v2-45db05405cb96248aff98ee07a565baa_1440w.jpg)

​                                                       *Transformer Encoder 编码句子信息*

**第三步**：将 Encoder 输出的编码信息矩阵 **C**传递到 Decoder 中，Decoder 依次会根据当前翻译过的单词 1~ i 翻译下一个单词 i+1，如下图所示。在使用的过程中，翻译到单词 i+1 的时候需要通过 **Mask (掩盖)** 操作遮盖住 i+1 之后的单词。

![img](LLM 篇.assets/v2-5367bd47a2319397317562c0da77e455_1440w.jpg)

​                                                                                  *Transofrmer Decoder 预测*

上图 Decoder 接收了 Encoder 的[编码矩阵](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=编码矩阵&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODQ5MDgwNTQsInEiOiLnvJbnoIHnn6npmLUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoxNjM0MjI5NzksImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.wOo8WjJ2ZJmZuLun_6ers8Zq6L5KAXRHFrrFIAU-gB8&zhida_source=entity) **C**，然后首先输入一个翻译开始符 "<Begin>"，预测第一个单词 "I"；然后输入翻译开始符 "<Begin>" 和单词 "I"，预测单词 "have"，以此类推。这是 Transformer 使用时候的大致流程，接下来是里面各个部分的细节。

#### 1.2 Transformer 的输入

Transformer 中单词的输入表示 **x**由**单词 Embedding** 和**位置 Embedding** （Positional Encoding）相加得到。

![img](LLM 篇.assets/v2-b0a11f97ab22f5d9ebc396bc50fa9c3f_1440w.jpg)

​																												*Transformer 的输入表示*

##### 1.2.1 单词 Embedding

单词的 Embedding 有很多种方式可以获取，例如可以采用 Word2Vec、Glove 等算法预训练得到，也可以在 Transformer 中训练得到。

##### 1.2.2 位置编码

**位置编码（Positional Encoding, PE）**是一个**与词嵌入维度相同的向量**，其目的是向模型注入关于词元在输入序列中绝对或相对位置的信息。它会与词元的词嵌入（Token Embedding）相加，然后一同输入到Transformer的底层。

###### 为什么它是必需的？

~~Transformer的核心机制——自注意力，在计算时处理的是一个集合（Set）而非序列（Sequence）。它本身不包含任何关于词元顺序的信息，是 <strong>置换不变（Permutation-invariant）</strong> 的。这意味着，如果打乱输入序列中词元的顺序，自注意力层的输出也会相应地被打乱，但每个词元自身的输出向量（在不考虑softmax归一化的情况下）是相同的。这显然不符合自然语言的特性，因为语序至关重要（例如“我打你”和“你打我”含义完全相反）。因此，必须通过一种外部机制，将位置信息显式地提供给模型，这就是位置编码的作用。~~

PE 可以通过训练得到，也可以使用某种公式计算得到。在 Transformer 中采用了后者。

###### **正弦/余弦位置编码（Sinusoidal Positional Encoding）：**

这是原始Transformer论文《Attention Is All You Need》中使用的方法。它使用不同频率的正弦和余弦函数来生成位置编码，其公式如下：
$$
PE_{(pos, 2i)} = \sin(pos / 10000^{2i/d_{\text{model}}})
$$

$$
PE_{(pos, 2i+1)} = \cos(pos / 10000^{2i/d_{\text{model}}})
$$

>**Transformer 的位置编码（Positional Encoding, PE）最终是一个矩阵。**
>
>- 每一行：表示一个 token（词）的位置编码向量
>- 每一列：表示这个向量的某一个维度
>- `pos` 表示第几个 token 的位置
>- `i` 表示位置编码向量中的第几个维度对， $$ i=0,1,2,\cdots,\frac{d_{model}}{2}-1 $$
>-  $d_{\text{model}}$ 是嵌入维度。
>
>* **优点：**
>  * **可外推性：** 能够处理比训练中最长序列还要长的序列。
>  * **相对位置信息：** 模型可以轻易地学习到相对位置关系，因为对于任何固定的偏移量 $k$ ， $PE_{pos+k}$ 都可以表示为 $PE_{pos}$​ 的一个线性函数，这使得模型更容易捕捉相对位置的依赖。
>
>所以一句话总结：
>
>> 位置编码 PE 是一个矩阵，每一行对应一个 token 的位置，每一列对应隐藏维度。公式中的 pos 决定“第几个 token”，i 决定“位置向量的第几个维度对”，sin/cos 给这个维度赋予不同频率的位置信息。

**正弦/余弦位置编码原理**

**1. 先看位置编码公式**

Transformer 的位置编码：
$$
PE_{(pos,2i)}=\sin(pos \cdot \omega_i)
$$

$$
PE_{(pos,2i+1)}=\cos(pos \cdot \omega_i)
$$

>这个是在说：
>
>**位置编码向量里面，每一个位置、每一个维度的值怎么算。**
>
>例如：
>
>$$ d_{model}=6 $$ 
>
>那么： 
>$$
>PE_{pos} = [ PE_{(pos,0)}, PE_{(pos,1)}, PE_{(pos,2)}, PE_{(pos,3)}, PE_{(pos,4)}, PE_{(pos,5)} ]
>$$
>展开： 
>$$
>= [ \sin(pos\omega_0), \cos(pos\omega_0), \sin(pos\omega_1), \cos(pos\omega_1), \sin(pos\omega_2), \cos(pos\omega_2) ]
>$$

其中：
$$
\omega_i=\frac{1}{10000^{2i/d_{\text{model}}}}
$$
也就是说，每两个维度是一组：
第 $2i$ 维：
$$
\sin(pos \cdot \omega_i)
$$
第 $2i+1$ 维：
$$
\cos(pos \cdot \omega_i)
$$
**2.假设位置移动 k**

现在看：
$$
PE_{pos+k}
$$
对于某一个频率 $\omega_i$：

正弦部分：
$$
\sin((pos+k)\omega_i) 
$$
展开： 
$$
=\sin(pos\omega_i+k\omega_i) 
$$
根据和角公式： 
$$
 \sin(A+B)=\sin A\cos B+\cos A\sin B 
$$
得到： 
$$
=\sin(pos\omega_i)\cos(k\omega_i) +\cos(pos\omega_i)\sin(k\omega_i)
$$
注意这里 $$ \sin(pos\omega_i) $$ 和 $$ \cos(pos\omega_i) $$  就是原来的位置编码里的两个维度。

所以： $$ PE_{pos+k} $$ 可以由： $$ PE_{pos} $$​ 里的两个值线性组合得到。



3.**举个具体例子**

假设**某两个维度**： 

位置 $pos$： 
$$
PE_{pos}=[\sin(pos\omega),\cos(pos\omega)]
$$
移动 $k$​​ 后：
$$
PE_{pos+k}
=
[\sin((pos+k)\omega),\cos((pos+k)\omega)]
$$
根据公式： 

第一项： 
$$
\sin((pos+k)\omega)
$$
等于： 
$$
\sin(pos\omega)\cos(k\omega) + \cos(pos\omega)\sin(k\omega) 
$$
第二项：
$$
\cos((pos+k)\omega)
$$
根据： $$ \cos(A+B)=\cos A\cos B-\sin A\sin B $$ 

得到： 
$$
= \cos(pos\omega)\cos(k\omega) - \sin(pos\omega)\sin(k\omega)
$$
写成矩阵： 
$$
\begin{bmatrix} \sin((pos+k)\omega) \\ \cos((pos+k)\omega) \end{bmatrix} = \begin{bmatrix} \cos(k\omega) & \sin(k\omega) \\ -\sin(k\omega) & \cos(k\omega) \end{bmatrix} \begin{bmatrix} \sin(pos\omega) \\ \cos(pos\omega) \end{bmatrix}
$$
右边那个矩阵： $$ \begin{bmatrix} \cos(k\omega) & \sin(k\omega) \\ -\sin(k\omega) & \cos(k\omega) \end{bmatrix} $$ 只和偏移量 $k$ 有关。



**4.固定矩阵 $$ A_k $$的大小（维度）也是固定的**

> 给定一个固定的位置偏移 k，新的位置编码可以通过一个固定矩阵  $$ A_k $$  对原位置编码进行线性变换得到。

**实际 Transformer：** 

假设：
$$
d_{model}=512
$$
位置编码： 
$$
PE_{pos}
$$
是：
$$
512维
$$
即： 
$$
PE_{pos}\in R^{512}
$$
那么： 
$$
A_k
$$
就是： 
$$
512\times512
$$
矩阵：
$$
PE_{pos+k}=A_kPE_{pos}
$$
![image-20260723041007866](LLM 篇.assets/image-20260723041007866.png)

>总结：
>
>$A_k$ 的维度是固定的。 对于 Transformer： $$ d_{model}=512 $$ 位置编码： $$ PE_{pos} $$ 是一个 512 维向量： $$ PE_{pos}\in R^{512} $$ 因此，用于表示位置偏移 $k$ 的线性变换矩阵： $$ A_k $$ 是一个： $$ 512\times512 $$ 的矩阵。  
>
>该矩阵由 256 个 $2\times2$ 的旋转矩阵组成。 每个 $2\times2$ 矩阵对应位置编码中的一对维度：
>$$
>[\sin(pos\omega_i),\cos(pos\omega_i)]
>$$
>这两个维度共同表示一个频率 $\omega_i$ 的位置信息。  不同的二维向量对应不同频率： $$ \omega_0,\omega_1,\omega_2,...,\omega_{255} $$ 因此整个位置编码由多种频率组合而成。  
>
>对于固定的位置偏移 $k$： 每个二维旋转矩阵只由： $$ k $$ 和对应频率： $$ \omega_i $$ 决定，而与具体的位置 $pos$ 无关。  因此： 
>$$
>PE_{pos+k}=A_kPE_{pos}
>$$
>表示： 任意位置经过相同偏移 $k$​ 后，都可以通过同一个线性变换得到。 这使模型能够更容易学习序列中的相对位置信息。

###### **可学习的绝对位置编码（Learned Absolute Positional Encoding）：**

这种方法将位置编码视为模型参数的一部分，通过训练学习得到。具体来说，会创建一个形状为 `(max_sequence_length, embedding_dimension)` 的位置编码矩阵。在处理序列时，根据每个词元的位置索引，从这个矩阵中查找对应的编码向量，并加到词嵌入上。BERT和GPT-2等模型采用了这种方式。

* <strong>优点：</strong> 模式更加灵活，可以让模型自己学习出最适合数据的位置表示。
* <strong>缺点：</strong> 无法泛化到超过预设 `max_sequence_length` 的长度。如果需要处理更长的序列，就需要对位置编码进行微调或采用其他策略。

将单词的词 Embedding 和位置编码相加，就可以得到单词的表示[向量](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=5&q=向量&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODQ5MDgwNTQsInEiOiLlkJHph48iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoxNjM0MjI5NzksImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6NSwiemRfdG9rZW4iOm51bGx9.jgkpx6x68tP6iJDmnJlmoBQh4L2QHRaa6vWIe1zi1wU&zhida_source=entity) **x**，**x** 就是 Transformer 的输入。

#####  1.2.3 Transformer 为什么需要位置编码

Self-Attention 本身不知道顺序。

比如：

```
我 爱 你
```

和

```
你 爱 我
```

输入 embedding：

```
我  -> [0.2, 0.5, ...]
爱  -> [0.1, 0.7, ...]
你  -> [0.8, 0.3, ...]
```

Attention 只看到三个向量，它不知道：

* 谁在前
* 谁在后
* 谁和谁相邻

所以需要给每个 token 加一个位置向量：

```
最终输入 = token embedding + position encoding
```

---

###### 1.PE 是一个矩阵

假设：

* 一句话长度：

```
seq_len = 4
```

有 4 个 token：

```
位置:
0    我
1    爱
2    你
3    好
```

假设：

```
d_model = 6
```

也就是每个 token 的隐藏向量长度是 6。

那么位置编码矩阵：

$$PE \in R^{4 \times 6}$$

长这样：

| 位置  | 维度0 | 维度1 | 维度2 | 维度3 | 维度4 | 维度5 |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| pos=0 | ?     | ?     | ?     | ?     | ?     | ?     |
| pos=1 | ?     | ?     | ?     | ?     | ?     | ?     |
| pos=2 | ?     | ?     | ?     | ?     | ?     | ?     |
| pos=3 | ?     | ?     | ?     | ?     | ?     | ?     |

每一行就是一个位置编码。

例如：

```
PE(pos=2)
=
[
0.91,
0.12,
0.76,
0.34,
0.65,
0.88
]
```

表示第 2 个 token 的位置。

---

###### 公式里的 pos 是什么？

看：

$$PE_{(pos,2i)}  
=  
\sin(pos / 10000^{2i/d_{model}})$$

这里：

**pos**

表示：

> 当前 token 的位置编号

例如：

句子：

```
我 爱 你
```

那么：

| token | pos  |
| ----- | ---- |
| 我    | 0    |
| 爱    | 1    |
| 你    | 2    |

所以：

计算第一个 token：

```
pos=0
```

第二个：

```
pos=1
```

第三个：

```
pos=2
```

---

###### i 是什么？

重点来了。

Transformer 的位置编码维度：

假设：

$$d_{model}=6$$

那么每个位置有 6 个数字：

```
位置编码:

[
x0,
x1,
x2,
x3,
x4,
x5
]
```

论文把维度两两分组：

```
维度:

0 1
2 3
4 5
```

每一组：

* 偶数维使用 sin
* 奇数维使用 cos

也就是：

| 维度 | 公式 |
| ---- | ---- |
| 0    | sin  |
| 1    | cos  |
| 2    | sin  |
| 3    | cos  |
| 4    | sin  |
| 5    | cos  |

---

所以：

**i=0**

对应：

```
维度0、1
```

计算：

偶数：

$$PE(pos,0)  
=  
sin(pos/10000^{0})$$

奇数：

$$PE(pos,1)  
=  
cos(pos/10000^{0})$$

---

**i=1**

对应：

```
维度2、3
```

计算：

$$PE(pos,2)  
=  
sin(pos/10000^{2/6})$$ $$PE(pos,3)  
=  
cos(pos/10000^{2/6})$$

---

**i=2**

对应：

```
维度4、5
```

计算：

$$PE(pos,4)  
=  
sin(pos/10000^{4/6})$$ $$PE(pos,5)  
=  
cos(pos/10000^{4/6})$$

---

###### 举个完整例子

假设：

```
d_model = 6

pos = 3
```

也就是：

第 3 个 token。

位置编码：

$$PE(3)  
=  
[  
PE(3,0),  
PE(3,1),  
PE(3,2),  
PE(3,3),  
PE(3,4),  
PE(3,5)  
]$$

分别计算：

---

第0维：

$$sin(3/10000^0)$$

---

第1维：

$$cos(3/10000^0)$$

---

第2维：

$$sin(3/10000^{2/6})$$

---

第3维：

$$cos(3/10000^{2/6})$$

---

第4维：

$$sin(3/10000^{4/6})$$

---

第5维：

$$cos(3/10000^{4/6})$$

最后得到：

```
pos=3:

[
0.141,
-0.99,
0.032,
0.999,
0.001,
1.000
]
```

（数值只是示意）

---

###### 为什么要用不同频率？

因为不同维度负责不同粒度的位置变化。

例如：

低维：

```
sin(pos)
```

变化快：

```
0
1
2
3
4
5
...
```

可以表示细粒度位置。

高维：

```
sin(pos/10000^0.9)
```

变化慢：

```
0.01
0.02
0.03
...
```

可以表示长距离位置。

类似：

```
维度0:

~~~~~~~ 快速波动


维度500:

________________ 缓慢变化
```

组合起来：

一个位置：

```
pos=10
```

会形成一个独特的 512 维向量。

---

###### 最终进入 Transformer 的形式

假设：

句子：

```
我 爱 你
```

token embedding：

$$X  
=  
\begin{bmatrix}  
E_我\\  
E_爱\\  
E_你  
\end{bmatrix}$$

大小：

$$3 \times 512$$

位置编码：

$$PE  
=  
\begin{bmatrix}  
PE_0\\  
PE_1\\  
PE_2  
\end{bmatrix}$$

也是：

$$3 \times 512$$

然后：

$$Input=X+PE$$

得到：

$$3\times512$$

送入 Transformer。

---

所以一句话总结：

> **位置编码 PE 是一个矩阵，每一行对应一个 token 的位置，每一列对应隐藏维度。公式中的 pos 决定“第几个 token”，i 决定“位置向量的第几个维度对”，sin/cos 给这个维度赋予不同频率的位置信息。**

### 2.Self-Attention（自注意力机制）

自注意力（Self-Attention）机制是Transformer模型的核心，它使得模型能够动态地衡量输入序列中不同单词之间的重要性，并据此生成每个单词的上下文感知表示。

![img](LLM 篇.assets/v2-f6380627207ff4d1e72addfafeaff0bb_1440w.jpg)

上图是论文中 Transformer 的内部结构图，左侧为 Encoder block，右侧为 Decoder block。红色圈中的部分为 **Multi-Head Attention**，是由多个 **Self-Attention**组成的，可以看到 Encoder block 包含一个 Multi-Head Attention，而 Decoder block 包含两个 Multi-Head Attention (其中有一个用到 Masked)。Multi-Head Attention 上方还包括一个 Add & Norm 层，Add 表示残差连接 (Residual Connection) 用于防止网络退化，Norm 表示 Layer Normalization，用于对每一层的激活值进行归一化。

因为 **Self-Attention**是 Transformer 的重点，所以我们重点关注 Multi-Head Attention 以及 Self-Attention，首先详细了解一下 Self-Attention 的内部逻辑。

#### 2.1 Self-Attention 结构

![img](LLM 篇.assets/v2-6444601b4c41d99e70569b0ea388c3bd_1440w.jpg)

​																										*Self-Attention 结构*

上图是 Self-Attention 的结构，在计算的时候需要用到矩阵**Q(查询),K(键值),V(值)**。在实际中，Self-Attention 接收的是输入(单词的表示向量x组成的矩阵X) 或者上一个 Encoder block 的输出。而**Q,K,V**正是通过 Self-Attention 的输入进行线性变换得到的。

#### 2.2 Q, K, V 的计算

对输入序列中的每个 token embedding，通过可学习矩阵$W^Q, W^K, W^V$ 进行线性变换，分别得到对应的 Query、Key、Value 向量。将所有 token 的 Query、Key、Value 向量按行堆叠后，形成 Query 矩阵 Q、Key 矩阵 K 和 Value 矩阵 V。

![img](LLM 篇.assets/v2-4f4958704952dcf2c4b652a1cd38f32e_1440w.jpg)

​																												*Q, K, V 的计算*

#### 2.3 Self-Attention 的输出

得到矩阵 Q, K, V之后就可以计算出 Self-Attention 的输出了，计算的公式如下：

![img](LLM 篇.assets/v2-9699a37b96c2b62d22b312b5e1863acd_1440w.jpg)

​                                                                                   *Self-Attention 的输出*

公式中计算矩阵**Q**和**K**每一行向量的内积，为了防止内积过大，因此除以$d_k$的平方根。**Q**乘以**K**的转置后，得到的矩阵行列数都为 n，n 为句子单词数，这个矩阵可以表示单词之间的 attention 强度。下图为**Q**乘以$K^{T}$ ，1234 表示的是句子中的单词。

![img](LLM 篇.assets/v2-9caab2c9a00f6872854fb89278f13ee1_1440w.jpg)

​                                                                           *Q乘以K的转置的计算*

得到$QK^{T}$之后，使用 Softmax 计算每一个单词对于其他单词的 attention 系数，公式中的 Softmax 是对矩阵的每一行进行 Softmax，即每一行的和都变为 1.

![img](LLM 篇.assets/v2-96a3716cf7f112f7beabafb59e84f418_1440w.jpg)

​                                                             *对矩阵的每一行进行 Softmax*

得到 Softmax 矩阵之后可以和**V**相乘，得到最终的输出**Z**。

![img](LLM 篇.assets/v2-7ac99bce83713d568d04e6ecfb31463b_1440w.jpg)

​                                                                            *Self-Attention 输出*

上图中 Softmax 矩阵的第 1 行表示单词 1 与其他所有单词的 attention 系数，最终单词 1 的输出 等于所有单词 i 的值$V_i$根据 attention 系数的比例加在一起得到，如下图所示：

![img](LLM 篇.assets/v2-27822b2292cd6c38357803093bea5d0e_1440w.jpg)

​	                                                                                              *Zi 的计算方法*

![image-20260724230548847](LLM 篇.assets/image-20260724230548847.png)

>单词1的输出是所有单词 Value 向量按照 Attention 权重进行加权求和得到的结果，其维度等于 Value 向量维度；在完整 Transformer 中，经过多头 Attention 拼接和线性映射后，恢复为模型隐藏维度$d_{model}$。



#### 2.4 补充理解

##### 缩放

**缩放**：将计算出的分数除以一个缩放因子 $\sqrt{d_k}$（ $d_k$​ 是K向量的维度）。这一步是为了在反向传播时获得更稳定的梯度，防止点积结果过大导致Softmax函数进入饱和区。

计算 Query 和 Key 的点积得到 attention 分数：
$$
QK^{T}
$$
点积本质上是：
$$
 q \cdot k=q_1k_1+q_2k_2+\cdots+q_{d_k}k_{d_k}
$$
 可以看到，**向量维度越大，参与相加的项越多，点积结果的数值可能越大。**为了避免点积结果过大，Transformer 在 Softmax 前进行缩放。

**为什么除以$\sqrt{d_k}$？**

因为：
$$
\mathrm{Var}(q \cdot k) \propto d_k
$$
所以除以$\sqrt{d_k}$可以把点积结果的波动范围缩小到稳定范围：
$$
\mathrm{Var}\left(\frac{q \cdot k}{\sqrt{d_k}}\right) \approx O(1)
$$

>$QK^T$ 的点积会随着向量维度 $d_k$ 增大而变大，因此 Transformer 除以 $\sqrt{d_k}$，把 attention 分数控制在稳定范围，避免 Softmax 进入饱和区，从而保证梯度稳定。



##### Softmax归一化

Softmax归一化：将缩放后的分数通过一个Softmax函数，使其转换为一组总和为1的概率分布。这些概率就是“注意力权重”，表示在当前位置，每个输入词元所占的重要性。
$$
\text{AttentionWeights} = \text{softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right)
$$



##### 加权求和

**加权求和**：最后，将得到的注意力权重与每个词元对应的V向量相乘并求和，得到最终的自注意力层输出。这个输出向量融合了整个序列的上下文信息，且权重由模型动态学习得到。
$$
\text{Output} = \text{AttentionWeights} \cdot V
$$

##### $\mathrm{Var}(q \cdot k) \propto d_k$ ？



#### 为什么比RNN更适合处理长序列？

1.  <strong>并行计算能力：</strong> 自注意力机制在计算时，可以一次性处理整个序列，计算所有位置之间的关联，是高度并行的。而RNN（包括LSTM、GRU）必须按照时间顺序依次处理每个词元，无法并行化，导致处理长序列时速度非常慢。
2.  <strong>解决长距离依赖问题：</strong> 在自注意力中，任意两个位置之间的交互路径长度都是O(1)，因为可以直接计算它们的注意力分数。而在RNN中，序列首尾两个词元的信息传递需要经过整个序列的长度，路径为O(N)，这极易导致梯度消失或梯度爆炸，使得模型难以捕捉长距离的依赖关系。



#### 过程图解

![image-20260724225555574](LLM 篇.assets/image-20260724225555574.png)



### 3. Multi-Head Attention

在上一步，我们已经知道怎么通过 Self-Attention 计算得到输出矩阵 Z，而 Multi-Head Attention 是由多个 Self-Attention 组合形成的，下图是论文中 Multi-Head Attention 的结构图。

**单头注意力在计算不同位置间关联时用到了加权平均，这在一定程度上影响了特征计算的准确性，因此要用多头注意力来抵消这种影响**。多头注意力允许模型在不同的位置关注信息的不同表征子空间。它通过并行执行多个上述的注意力函数，并将它们的结果拼接起来再通过一个线性变换来实现。具体来说：

对于每一个 head，都有独立的参数矩阵：

**查询矩阵： **
$$
W_Q^i \in \mathbb{R}^{d_{model} \times d_k}
$$
**键矩阵： **
$$
W_K^i \in \mathbb{R}^{d_{model} \times d_k}
$$
**值矩阵： **
$$
W_V^i \in \mathbb{R}^{d_{model} \times d_v}
$$


用于将输入转换成查询、键和值。每个头的输出是按照上述公式计算出来的。所有头的输出会被拼接在一起，并通过一个线性变换 $W_O$ 来整合这些输出。这个最终的**线性变换矩阵 $W_O$ 的尺寸是 $ \mathbb{R}^{h d_v \times d_{model}}$**，其中 $h$ 是头的数量。

多头注意力的公式可以表示为：

$$
\text{MultiHead}(Q,K,V)=\text{Concat}(head_1,\ldots,head_h)W_O
$$

其中：

$$
head_i=\text{Attention}(QW_Q^i,KW_K^i,VW_V^i)
$$
![img](LLM 篇.assets/v2-b0ea8f5b639786f98330f70405e94a75_1440w.jpg)

​                                                                               *Multi-Head Attention*

从上图可以看到 Multi-Head Attention 包含多个 Self-Attention 层，首先将输入**X**分别传递到 h 个不同的 Self-Attention 中，计算得到 h 个输出矩阵**Z**。

#### 3.1 Multi-Head Attention 单头计算图解 

单头注意力包含 $W_Q$、$W_K$、$W_V$，产生一个 output。 多头注意力则包含 $n$ 个 $W_Q$、$W_K$、$W_V$，这些参数的权重不共享，产生 $n$​ 个 output。

![image-20260725092528185](LLM 篇.assets/image-20260725092528185.png)

​                                                             *Multi-Head Attention单头拼接计算过程*

#### 3.2 Multi-Head Attention 多头计算过程图解

 在多头注意力机制（Multi-Head Attention）中，各个头产生的输出确实被拼接在一起，并且对拼接后的结果再次进行线性变换（projection），以得到最终的输出。这个过程确保了模型可以综合不同表征子空间的信息，同时保持输出维度的一致性。

![image-20260725092838486](LLM 篇.assets/image-20260725092838486.png)

​                                                                         *Multi-Head Attention多头计算过程*



**Multi-Head Attention多头拼接计算过程**

![image-20260725093129830](LLM 篇.assets/image-20260725093129830.png)

​                                                                     Multi-Head Attention多头拼接计算过程



### 4. MQA（Multi Query Attention）

MQA（Multi-Query Attention）是一种优化的注意力机制，旨在减少 Transformer 模型中多头注意力（Multi-Head Attention，MHA）计算的参数量和计算成本。与标准的 MHA 不同，MQA 通过让所有的注意力头共享同一个键（Key，K）和值（Value，V）线性映射矩阵来实现这一点，而查询（Query，Q）矩阵仍然为每个头独立设置。

---

> 在传统的多头注意力机制中，每个注意力头都有独立的 $W_Q^i$、$W_K^i$、$W_V^i$​​ 矩阵用于将输入转换成查询、键和值。这导致了参数量的增长，尤其是在头数较多的情况下。MQA 简化了这一过程：
>
> - **共享的键和值矩阵：对于每一层，所有注意力头共享同一个键 $K$ 和值 $V$ 线性映射矩阵，即：**$W_K$ **和** $W_V$​
>
>     这意味着每层只有一个 $W_K$ 和 $W_V$ 矩阵。
>
> - **独立的查询矩阵：查询矩阵 $Q$ 仍然为每个头独立设置，即：$W_Q^1, W_Q^2, \ldots, W_Q^h$，其中 $h$ 是头的数量。**

这种设计减少了模型的参数量，因为键和值的线性变换不再依赖于头的数量，从而降低了计算复杂度，特别是在大规模模型或长序列上时性能提升更为显著。

#### 4.1 Multi Query Attention 计算过程

MQA计算公式表示为：

$$
\text{MultiQueryAttention}(Q,K,V)=\text{Concat}(head_1,\ldots,head_h)W_O
$$

其中：

$$
head_i=\text{Attention}(QW_Q^i,KW_K,VW_V)
$$

这里的 $W_Q^i$ 是第 $i$ 个头的查询矩阵，而 $W_K$ 和 $W_V$ 是该层所有头共享的键和值矩阵。

---

> 参数矩阵的维度
>
> - 查询矩阵 $W_Q^i$ 的维度为 $d_{model}\times d_k$​， 每个头都有自己独立的查询矩阵。
> - 共享的键矩阵 $W_K$ 和值矩阵 $W_V$ 的维度分别为$W_K \in \mathbb{R}^{d_{model}\times d_k}$ 和 $W_V \in \mathbb{R}^{d_{model}\times d_v}$， 这两个矩阵在整个层内被所有头共享。



#### 4.2Multi Query Attention 计算过程图解

![image-20260725101941789](LLM 篇.assets/image-20260725101941789.png)

​                                                                         *Multi Query Attention 计算过程*

MQA只让 $Q$ 保留了原始多头的性质（每个 Head 存在不同的转换），可以大大减少 $K$ 和 $V$ 矩阵的参数量以及 KV Cache 的显存占用，提升推理速度。但是可能会带来精度上的损失。广泛应用于大规模语言模型（LLMs），如 ChatGLM2。        







### 5.GQA（Group Query Attention） 

GQA（Grouped Query Attention）是一种在 MQA 基础上进一步优化的注意力机制，旨在平衡参数量减少和模型性能之间的关系。通过将查询（Query，Q）进行分组，并让每个组内的所有头共享同一组键（Key，K）和值（Value，V），GQA 试图保留多头注意力机制的优势，同时减少计算成本和显存占用。 

> GQA（Group Query Attention）实现步骤： 
>
> 1. **查询分组：**将所有的查询 $Q$ 分成若干个组，每个组内的查询共享一组键 $K$ 和值 $V$。
> 2.  **共享键和值矩阵：**每个组内共享同一个键矩阵 $$ W_K^g $$ 和值矩阵$$ W_V^g $$ 其中 $g$ 表示组编号。 
> 3.  **独立的查询矩阵：**每个组内的查询仍然有自己的查询矩阵： $$ W_Q^{i,g} $$ 这里 $i$ 表示组内的头编号，$g$ 表示组编号。 
> 4. **输出拼接和投影：**所有组的输出被拼接在一起，并通过线性变换矩阵$$ W_O $$​​ 进行投影。

#### 5.1 Group Query Attention 计算过程及图解

GQA的计算公式表示为：

$$
\text{GroupedQueryAttention}(Q,K,V)=\text{Concat}(group_1,\ldots,group_G)W_O
$$

其中：

$$
group_g=\text{Concat}(head_{1,g},\ldots,head_{h_g,g})
$$

$$
head_{i,g}=\text{Attention}(QW_Q^{i,g},KW_K^g,VW_V^g)
$$

这里的 $W_Q^{i,g}$ 是第 $g$ 组中第 $i$ 个头的查询矩阵，而 $W_K^g$ 和 $W_V^g$ 是该组内所有头共享的键和值矩阵。      

![image-20260725104701999](LLM 篇.assets/image-20260725104701999.png)





### 6.MLA( Multi-head Latent Attention)

略

### 7.四种 attention 机制的对比 

MQA 这种共享一个 $K$、$V$，只有 $Q$​ 是独立的，大幅减少了参数数量，带来推理加速的同时会造成模型性能损失，且在训练过程中使得模型变得不稳定，从而提出了 GQA。它将 Query 进行分组，每个组内共享一组 Key、Value。 

#### 7.1 四种 attention 计算过程 

当 GQA 的分组数 $N$ 等于 $1$ 时，GQA 等价于 MQA，所有的查询 $Q$ 共享一组 $K$、$V$。 当 GQA 的分组数 $N$ 等于查询 $Q$ 的数量时，GQA 等价于 MHA，每一个 $Q$ 对应自己的 $K$、$V$​ 矩阵。

![image-20260725111022403](LLM 篇.assets/image-20260725111022403.png)

## Encoder 结构

![img](LLM 篇.assets/v2-0203e83066913b53ec6f5482be092aa1_r.jpg)

上图红色部分是 Transformer 的 Encoder block 结构，可以看到是由 Multi-Head Attention, **Add & Norm, Feed Forward, Add & Norm** 组成的。刚刚已经了解了 Multi-Head Attention 的计算过程，现在了解一下 Add & Norm 和 Feed Forward 部分。

### 4.1 Add & Norm

Add & Norm 层由 Add 和 Norm 两部分组成，其计算公式如下：

![img](LLM 篇.assets/v2-a4b35db50f882522ee52f61ddd411a5a_1440w.jpg)

其中 **X**表示 Multi-Head Attention 或者 Feed Forward 的输入，MultiHeadAttention(**X**) 和 FeedForward(**X**) 表示输出 (输出与输入 **X** 维度是一样的，所以可以相加)。

>所以这里的“维度一样”主要是指：
>
>输入和输出的隐藏维度 $d_{model}$ 相同，这样才能和输入 **X** 做残差连接。
>$$
>X \in R^{n \times d_{model}}
>$$
>
>$$
>MultiHeadAttention(X),\ FeedForward(X) \in R^{n \times d_{model}}
>$$

**Add**指 **X**+MultiHeadAttention(**X**)，是一种残差连接，通常用于解决多层网络训练的问题，可以让网络只关注当前差异的部分，在 [ResNet](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=ResNet&zhida_source=entity) 中经常用到：

![img](LLM 篇.assets/v2-4b3dde965124bd00f9893b05ebcaad0f_1440w.jpg)

​                                                                                     *残差连接*

**Norm**指 Layer Normalization，通常用于 RNN 结构，Layer Normalization 会将每一层神经元的输入都转成均值方差都一样的，这样可以加快收敛。

### 4.2 Feed Forward

Feed Forward 层比较简单，是一个两层的[全连接层](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=全连接层&zhida_source=entity)，第一层的激活函数为 Relu，第二层不使用激活函数，对应的公式如下。

![img](LLM 篇.assets/v2-47b39ca4cc3cd0be157d6803c8c8e0a1_1440w.jpg)

​                                                                                         *Feed Forward*

**X**是输入，Feed Forward 最终得到的输出矩阵的维度与**X**一致。

### 4.3 组成 Encoder

**通过上面描述的 Multi-Head Attention, Feed Forward, Add & Norm 就可以构造出一个 Encoder block，Encoder block 接收输入矩阵 $X_{(n \times d)}$ ，并输出一个矩阵 $O_{(n \times d)}$。通过多个 Encoder block 叠加就可以组成 Encoder。**

第一个 Encoder block 的输入为句子单词的表示向量矩阵，后续 Encoder block 的输入是前一个 Encoder block 的输出，最后一个 Encoder block 输出的矩阵就是**编码信息矩阵 C**，这一矩阵后续会用到 Decoder 中。

![img](LLM 篇.assets/v2-45db05405cb96248aff98ee07a565baa_1440w-1784985557020-9.jpg)

​                                                                *Encoder 编码句子信息*



## Decoder 结构

![img](LLM 篇.assets/v2-f5049e8711c3abe8f8938ced9e7fc3da_1440w.jpg)

​                                                                            *Transformer Decoder block*

上图红色部分为 Transformer 的 Decoder block 结构，与 Encoder block 相似，但是存在一些区别：

- 包含两个 Multi-Head Attention 层。
- 第一个 Multi-Head Attention 层采用了 Masked 操作。
- 第二个 Multi-Head Attention 层的**K, V**矩阵使用 Encoder 的**编码信息矩阵C**进行计算，而**Q**使用上一个 Decoder block 的输出计算。
- 最后有一个 Softmax 层计算下一个翻译单词的概率。

### 第一个 Multi-Head Attention

Decoder block 的第一个 Multi-Head Attention 采用了 Masked 操作，因为在翻译的过程中是顺序翻译的，即翻译完第 i 个单词，才可以翻译第 i+1 个单词。通过 Masked 操作可以防止第 i 个单词知道 i+1 个单词之后的信息。下面以 "我有一只猫" 翻译成 "I have a cat" 为例，了解一下 Masked 操作。

下面的描述中使用了类似 Teacher Forcing 的概念，不熟悉 Teacher Forcing 的童鞋可以参考以下上一篇文章Seq2Seq 模型详解。在 Decoder 的时候，是需要根据之前的翻译，求解当前最有可能的翻译，如下图所示。首先根据输入 "<Begin>" 预测出第一个单词为 "I"，然后根据输入 "<Begin> I" 预测下一个单词 "have"。

![img](LLM 篇.assets/v2-4616451fe8aa59b2df2ead30fa31dc98_1440w.jpg)

​                                                                                               *Decoder 预测*

Decoder 可以在训练的过程中使用 Teacher Forcing 并且并行化训练，即将正确的单词序列 (<Begin> I have a cat) 和对应输出 (I have a cat <end>) 传递到 Decoder。那么在预测第 i 个输出时，就要将第 i+1 之后的单词掩盖住，**注意 Mask 操作是在 Self-Attention 的 Softmax 之前使用的，下面用 0 1 2 3 4 5 分别表示 "<Begin> I have a cat <end>"。**

**第一步：**是 Decoder 的输入矩阵和 **Mask** 矩阵，输入矩阵包含 "<Begin> I have a cat" (0, 1, 2, 3, 4) 五个单词的表示向量，**Mask** 是一个 5×5 的矩阵。在 **Mask** 可以发现单词 0 只能使用单词 0 的信息，而单词 1 可以使用单词 0, 1 的信息，即只能使用之前的信息。

![img](LLM 篇.assets/v2-b26299d383aee0dd42b163e8bda74fc8_1440w.jpg)

​                                                                        *输入矩阵与 Mask 矩阵*

第二步：接下来的操作和之前的 Self-Attention 一样，通过输入矩阵X计算得到Q,K,V矩阵。然后计算Q和 $K^T$ 的乘积 $QK^T$​。

![img](LLM 篇.assets/v2-a63ff9b965595438ed0c0e0547cd3d3b_1440w.jpg)

​                                                                       *Q乘以K的转置*

第三步：在得到 $QK^T$​ 之后需要进行 Softmax，计算 attention score，我们在 Softmax 之前需要使用Mask矩阵遮挡住每一个单词之后的信息，遮挡操作如下：

![img](LLM 篇.assets/v2-35d1c8eae955f6f4b6b3605f7ef00ee1_1440w.jpg)

​                                                                              *Softmax 之前 Mask*

得到 Mask $QK^T$ 之后在 Mask $QK^T$ 上进行 Softmax，每一行的和都为 1。但是单词 0 在单词 1, 2, 3, 4 上的 attention score 都为 0。

第四步：使用 Mask $QK^T$ 与矩阵 V相乘，得到输出 Z，则单词 1 的输出向量 $z_1$ 是只包含单词 1 信息的。
![img](https://picx.zhimg.com/v2-58f916c806a6981e296a7a699151af87_1440w.jpg)

​                                                                        *Mask 之后的输出*

第五步：通过上述步骤就可以得到一个 Mask Self-Attention 的输出矩阵 $Z_i$ ，然后和 Encoder 类似，通过 Multi-Head Attention 拼接多个输出 $Z_i$  然后计算得到第一个 Multi-Head Attention 的输出 $Z$，$Z$ 与输入 $X$ 维度一样。



### 第二个 Multi-Head Attention

Decoder block 第二个 Multi-Head Attention 变化不大， 主要的区别在于其中 Self-Attention 的 **K, V**矩阵不是使用 上一个 Decoder block 的输出计算的，而是使用 **Encoder 的编码信息矩阵 C** 计算的。

根据 Encoder 的输出 **C**计算得到 **K, V**，根据上一个 Decoder block 的输出 **Z** 计算 **Q** (如果是第一个 Decoder block 则使用输入矩阵 **X** 进行计算)，后续的计算方法与之前描述的一致。

这样做的好处是在 Decoder 的时候，每一位单词都可以利用到 Encoder 所有单词的信息 (这些信息无需 **Mask**)。

### Softmax 预测输出单词

Decoder block 最后的部分是利用 Softmax 预测下一个单词，在之前的[网络层](https://zhida.zhihu.com/search?content_id=163422979&content_type=Article&match_order=1&q=网络层&zhida_source=entity)我们可以得到一个最终的输出 Z，因为 Mask 的存在，使得单词 0 的输出 Z0 只包含单词 0 的信息，如下：

![img](LLM 篇.assets/v2-335cfa1b345bdd5cf1e212903bb9b185_1440w.jpg)

​                                                                           *Decoder Softmax 之前的 Z*

Softmax 根据输出矩阵的每一行预测下一个单词：

![img](LLM 篇.assets/v2-0938aa45a288b5d6bef6487efe53bd9d_1440w.jpg)

​                                                                 *Decoder Softmax 预测*

这就是 Decoder block 的定义，与 Encoder 一样，Decoder 是由多个 Decoder block 组合而成。



## KV Cache

[深入理解 KV-Cache - 大模型推理的核心加速技术](https://wilsonwu.me/blog/2025/kv-cache-in-llm/)

随着大语言模型（LLM）规模不断增长，推理成本也随之飙升。为了让模型在响应用户请求时更快、更经济地运行，各类优化技术不断涌现。其中，**KV-Cache（Key-Value Cache）**是目前最关键、影响最深远的推理加速机制之一，被所有主流的推理框架（如 vLLM、TensorRT-LLM、LLama.cpp、llm-d、OpenAI Triton Transformer Engine 等）广泛使用。

### KV-Cache 是什么？

首先，其工作原理架构图如下：

![架构图](LLM 篇.assets/1-architecture.png)

理解起来非常简单，单其中细节结合复杂度计算还是略有抽象。

KV-Cache，全称 **Key-Value Cache**，是在 LLM 推理过程中对 Transformer 解码器中**自注意力层（Self-Attention）**的中间结果进行缓存的一种方法。

Transformer 的自回归生成方式决定了： **每生成一个新 token，就需要重新计算它与全部历史 token 的注意力关系**。

如果每次都把过去所有 token 的 Key 和 Value 重新计算一遍，其计算量是：

> 推理时间复杂度：O(n²) （n 为上下文长度）

为了避免重复计算，KV-Cache 会在每步生成 token 时，将计算后的 **Key（K）** 和 **Value（V）** 保存下来，这样之后再生成下一步 token 时就能直接引用过去的 Key/Value，而无需重新计算历史部分。

这让时间复杂度大幅下降为：

> 推理时间复杂度：O(n)

所以简单总结一下，KV-Cache 是在自回归解码中缓存历史 token 的 Key/Value，让后续生成直接复用过去的注意力结果，从而把时间复杂度从 O(n²) 降到 O(n)，节省的成本非常巨大。



### KV-Cache 是如何工作的？

以当前主流的解码流程为例，我们以是否使用 KV-Cache 来做一个简单的对比。

#### 如果没有 KV-Cache

每次生成新 token 时需要做：

1. 重新对全部历史序列做 embedding
2. 通过所有 Transformer 层重新计算 K/V
3. 根据新 token 与整个序列做自注意力
4. 得到新 token 并输出

具体流程图示如下：

![不使用 KV-Cache](LLM 篇.assets/2-without-kv-cache.png)

#### 使用 KV-Cache

第一次推理（prefill 阶段）：

1. 对所有提示词（prompt tokens）进行计算
2. 计算所有 K/V，并将它们保存在 KV 缓存中

之后的每一步（decode 阶段）：

1. 只计算当前新 token 的 K/V
2. 与缓存中的 K/V 做注意力，得到结果
3. 写回缓存

这样，每生成一个 token 的计算量从**全序列规模 → 近似常数开销**，越长的上下文收益越大。

其流程图示如下：

![使用 KV-Cache](LLM 篇.assets/3-with-kv-cache.png)



### KV-Cache 带来了什么影响？

#### 推理速度提升数十倍

对长上下文模型（8k / 32k / 128k tokens）尤其明显。

例如对 32k token 的上下文，如果没有 KV-Cache：

- 每生成一个 token 需要遍历 32k 历史序列
- 整体是 *重复 32k 次的巨量工作*

有了 KV-Cache：

- 历史部分只算一次
- 单步 decode 的开销变成固定量（几百维 KV 与注意力点积）

通常加速比可达到：

> 10 倍至 100 倍不等 （取决于提示词长度、模型大小、硬件）

#### 推理成本大幅降低

大模型推理的成本主要由这几个部分构成：

- **算力**（矩阵乘法）
- **显存占用**
- **带宽**

KV-Cache 能大幅降低重复算力，因此成本成比例下降。

这就是为什么：

- OpenAI、Anthropic、Meta、Mistral
- 各大云推理服务
- 所有开源推理框架

都把 KV-Cache 作为默认推理方式。

#### 但是显存占用会变高

KV 是每一层每个 token 的结果，因此缓存本身需要**巨大的显存**。

例如： Llama-3 70B 模型，64K context，对 1 token 的 KV 占用约 1.5KB。

那么 64K token 就是：**约 96MB × 80 层 ≈ 7.6GB 的 KV-cache**

当你看到推理卡 80GB 显存不够时，很大概率是因为 KV 满了，而不是模型权重太大。

#### 影响了模型的架构设计与行业走向

KV-Cache 的重要性高到：**模型架构都围绕它做优化。**

包括：

- Mistral 的**滑动窗口注意力（SWA）**
- Meta 的 **Grouped Query Attention（GQA）**
- DeepSeek 的 **Multi-head Latent Attention（MLA）**
- Google 的 **Multi-Query Attention（MQA）**

这些技术主要目的之一就是：

> 降低 KV-Cache 的显存占用

因为 Q 数少 → KV 也变少 → 显存压力更低。

所以 **KV-Cache 不只是一个技术点，而是影响模型架构的重要因素**。



### KV-Cache 的最佳实践

#### 尽量减少上下文长度

上下文越长 → KV 越大 → 推理越慢、越贵。

如果业务允许：

- 使用 **RAG** 把长文档拆分
- 使用**压缩模型（compressor）**
- 使用**短指令模板**
- 引导模型减少“废话”输出

#### 选择合适的 Attention 架构

如果你在自定义模型、微调或部署模型：

- 需要高效推理 → 优选 **MQA / GQA / MLA**
- 需要兼容许多任务 → MHA 仍然最稳定

#### 使用专业推理框架

优先使用：

- **vLLM（商业与开源界第一推理框架）**
- **TensorRT-LLM（NVIDIA 官方）**
- **llm-d（中小型企业的极高性价比框架）**
- **OpenAI Triton 引擎（内部使用）**

它们都在 KV-Cache 管理上有深度优化。

#### 多卡推理时注意 KV 分布问题

KV 会引发：

- **多卡同步开销**
- **all-reduce 带宽瓶颈**

部分框架提供：

- KV sharding
- pipeline parallelism
- block-sparse KV sharing

合理设置可显著提升吞吐量。

#### 批处理策略影响巨大

适配 KV-Cache 的批处理策略包括：

- continuous batching
- block scheduling
- prompt+decode 混合 batch

正确 batching 通常带来 **>10× 吞吐提升**。

### KV-Cache 的未来展望

未来几年，模型推理的发展趋势中，KV-Cache 会继续是核心机制。

我们将看到：

- 更轻量的 KV 结构（更少头、更小维度）
- 更智能的 KV 重用机制
- KV 的跨请求共享（multi-query serving）
- 部分 KV 的 CPU/NVMe offload
- Push-down Attention（降低 GPU 带宽要求）

总之，KV-Cache 将继续驱动推理性能提升，是 LLM 推理领域最重要的基础设施之一。



## 请比较一下几种常见的 LLM 架构，并说明它们各自最擅长的任务类型

### Transformer

![img](LLM 篇.assets/v2-8c9d44a303fc62b6ec715e8ff641a78b_1440w.jpg)

1. **编码器-解码器结构**：Transformer模型采用编码器-解码器结构，其中编码器负责将输入序列映射为连续表示，解码器则根据编码器的输出逐步生成输出序列。编码器和解码器是Transformer模型中的两个关键组件，它们在序列转导任务中扮演着不同的角色：编码器负责将输入序列编码为连续表示，而解码器则根据编码器的输出和先前生成的部分序列来生成输出序列。
2. **编码器层和解码器层**：Transformer模型包含编码器层和解码器层。每个编码器层由两个子层组成：一个多头自注意力机制和一个全连接前馈网络。解码器层也类似，但在[多头自注意力机制](https://zhida.zhihu.com/search?content_id=240310861&content_type=Article&match_order=2&q=多头自注意力机制&zhida_source=entity)中会引入一个编码器-解码器注意力机制，用于捕捉输入和输出序列之间的依赖关系。编码器层负责将输入序列转换为连续的表示，捕捉输入序列中的信息并建模其特征。通过堆叠多个编码器层，模型可以逐步提取输入序列中的特征，并利用自注意力机制来捕捉输入序列中各个位置之间的依赖关系。解码器层负责根据编码器的输出和先前生成的部分序列来逐步生成输出序列。解码器通过堆叠多个解码器层，利用编码器的输出和自身的注意力机制来生成输出序列。解码器还包含编码器-解码器注意力机制，用于捕捉输入和输出序列之间的依赖关系。

我们这里提到了编码器和解码器、编码器层和解码器层的作用，至于具体结构参照上图，那我们首先应该理解的就是编码器是对输入序列做建模表示，解码器是对编码器的输出及之前生成的序列来做解码输出。

### 大模型结构

下面这张图是一个大模型的一个分布树，纵轴代表大模型的发布年份和大模型输入token数，这个图很有代表性，每一个分支代表不同的模型架构，今天以图中根系标注的三大类展开：Encoder-only、Encoder-Decoder、Decoder-only;我们分别来看一下这几个架构的特点和原理吧。

![img](LLM 篇.assets/v2-eec0cdfbd54de464fb289b74e6889c19_1440w.jpg)



#### Encoder-Only 架构

Encoder-only是以Bert为代表的模型及其衍生优化版本为主，那就以Bert为例来学习Encoder-only架构；
BERT（Bidirectional Encoder Representations from Transformers）是一种在自然语言处理（NLP）领域引起巨大轰动的预训练语言模型，由Google于2018年提出。其核心原理是**结合了Transformer架构和双向语言模型预训练策略**，使得模型能够更好地理解和表征文本数据。BERT的出现在NLP领域引起了巨大的轰动，因为它在各种NLP任务上取得了巨大成功，同时也开创了一种新的预训练模型的范式。源码：[https://github.com/google-research/bert](https://link.zhihu.com/?target=https%3A//github.com/google-research/bert)。

1. **BERT原理**：
   1. Transformer 架构：BERT基于Transformer模型。BERT仅使用编码器部分，因为它主要用于处理单向输入数据。Transformer的核心是自注意力机制（Self-Attention），**它允许模型在编码输入序列时同时考虑序列中的所有位置，而无需将注意力限制在固定大小的窗口内**。自注意力机制使得模型能够在不同位置之间建立关联，从而更好地理解上下文信息。
   2. 双向语言模型预训练：传统的语言模型通常是单向的，只能从左到右或者从右到左预测文本。**而BERT采用了双向语言模型预训练的策略，能够同时考虑文本序列中的左右上下文信息**。这使得模型在预训练阶段能够更好地捕捉文本的语义和语法信息，从而生成更加丰富的语言表示。
      1. 在双向语言模型预训练中，BERT采用了两个任务：
         1. Masked Language Model（MLM）：**模型随机地掩盖输入文本中的一部分词语，然后尝试预测这些被掩盖的词语。这使得模型必须基于其上下文来推断被掩盖的词语，从而学习双向的语言表示。**
         2. Next Sentence Prediction（NSP）：模型接受两个句子作为输入，并尝试预测这两个句子是否是连续的。这个任务可以帮助模型理解文本之间的逻辑关系和连贯性。

![img](LLM 篇.assets/v2-52a02eee4c1d93cceac6938c6a0e45b6_1440w.jpg)

左侧的预训练部分展示了BERT如何在无标签数据上进行训练。图中显示了两个任务：下一个句子预测（Next Sentence Prediction, NSP）和掩码语言模型（Masked Language Model, Mask LM）。在NSP任务中，模型预测两个文本片段是否在原始文档中相邻。在Mask LM任务中，BERT预测输入中被[MASK]标记替换的原始词语。

右侧显示了微调过程。在这里，预训练的BERT模型被调整以适应特定任务，本例中涉及对一个名为SQuAD的数据集进行问答。在BERT之上添加了一个额外的层，通常是一个分类器，用于特定任务。在微调期间，所有参数都一起微调。

两个图中都显示了输入中的标记：[CLS]，用于分类任务的特殊标记，以及[SEP]，用于分隔段落的标记。此外，在右侧，带有注释的箭头表示模型预测文本中答案范围的“开始”和“结束”。

3. 多层嵌套结构：BERT使用了多层Transformer编码器，**并且在这些层之间引入了不同的任务特定变换**。每个编码器由多头自注意力层和前馈神经网络层组成，以及一些残差连接和层归一化层。这种多层嵌套结构使得BERT能够学习多层次、多粒度的语言表示，适应不同任务的需求。

一句话总结，BERT核心原理：使用多层嵌套的Transformer的编码器来处理输入序列，使用双向语言模型预训练策略进行掩码预测；到这里大家可能想问了，为什么要用这种框架，Bert开始的时候只是希望能够用这个框架能够学习语言的语法规则，针对主要是文本分类、问答等任务，所以只需要使用Transformer的编码器能够实现文本的语义理解就可以了，不需要生成序列。
搞清楚了Bert原理，那为什么说BERT属于Encoder-only模型？
很简单，因为它**只使用了Transformer模型中的编码器部分，而没有使用解码器**。在Transformer模型中，编码器负责将输入序列转换为上下文感知的表示，而解码器则负责生成输出序列。BERT使用了编码器。只使用编码器最主要的原因：BERT的预训练目标是**通过掩盖部分输入来预测其他部分，或者预测两个句子之间的关系**，这些任务并不涉及到生成输出序列，因此不需要解码器。上面我们已经说Transformer的解码器的作用，不懂请回看；
那分析一下BERT的存在的问题：

1. **大型模型需求**：BERT及其衍生模型通常需要大量的计算资源和存储空间来训练和部署。大型模型需要更多的参数和更多的计算，这会导致训练和推理的成本较高，限制了它们在资源有限的环境中的应用。
2. **上下文理解能力有限**：尽管BERT在一定程度上能够理解输入文本的上下文信息，但它并不能完全解决长距离依赖问题。在处理长文本或长距离依赖关系的任务时，BERT可能会出现性能下降或信息丢失的情况。
3. **缺乏对抗性**：BERT模型在处理对抗性样本（adversarial examples）时表现较差，容易受到攻击和干扰。对抗性样本是经过微小修改的输入，对人类来说很难察觉，但可以导致模型输出错误。
4. **预训练数据偏差**：BERT模型通常是在大规模通用文本数据上进行预训练的，这可能导致模型在某些特定领域或任务上的表现不佳。如果任务领域与预训练数据的领域不匹配，模型可能需要更多的微调或领域适应。
5. **无法处理多个任务的动态适应性：**尽管BERT模型可以通过微调来适应特定的下游任务，但在处理多个任务时，模型的动态适应性有限。因此，对于多任务学习或者需要动态适应不同任务的场景，BERT可能并不是最佳选择。
6. **固定长度输入限制**：BERT模型的输入长度有限，通常为512个token，这限制了模型处理较长文本的能力。对于长文本或长句子，需要进行截断或分段处理，可能会损失一些上下文信息。

>* **结构：**由多个Transformer Encoder层堆叠而成。
>* **核心机制：**双向自注意力机制。在处理序列中的任何一个词元时，模型都可以同时关注到它左边和右边的所有词元。这使得模型能够获得非常丰富的上下文表示。
>* 最擅长的任务类型：自然语言理解 (NLU)。
>  * <strong>具体任务：</strong>
>    * 分类任务： 情感分析、文本分类。
>    * 序列标注：命名实体识别 (NER)。
>    * 句子关系判断： 自然语言推断 (NLI)。
>    * 完形填空： 像BERT的Masked Language Model (MLM) 预训练任务本身。
>  * 原因：这些任务的核心是<strong>理解</strong>输入文本的深层含义，而双向上下文对于准确理解至关重要。这类模型的输出通常是固定的标签或类别，而非自由生成的长文本。



#### Encoder-Decoder

从上面的图中我们可以看到Encoder-Decoder架构的模型有T5、GLM等，为了能够让更多的人看懂，我们就以清华大学的GLM为例来继续，GLM的全称**基于自回归空白填充预训练框架（**General Language Model Pretraining with Autoregressive Blank Infilling**），这个框架的思路，结合BERT的思路，从输入文本中随机地空白出连续的跨度的token，并按照自回归预训练的思路，训练模型来依次重建这些跨度。GLM掩码的不是一个单词或是一个实体，而是一个句子片段。**

![img](LLM 篇.assets/v2-84d9c981703edf9a44ba8bb1a0244cb1_1440w.jpg)


这里出现了一个新的名词：自回归；自回归的核心思想是**使用当前时刻的输入来预测下一个时刻的输出**。在语言模型中，输入通常是文本序列的前一部分，而输出是下一个词或字符。大家只需要理解这一点就可以了，下一章节我们会详细说明；
GLM最初是为了能够NLP的任务更加全面的解决。所以借鉴了BERT的思路对输入的表示进行建模，同时采用了解码器构建模型的生成能力，主要的做法，请结合下图阅读：

![img](LLM 篇.assets/v2-41f170ac089227cecac3f5b00939f9b9_1440w.jpg)



- 对输入X进行随机采样，采样结果进行Mask替换，如图：原始文本x1x2x3x4x5x6中的 x3 和 x5x6 分别用 [M]替换，变成 x1x2[M]x4[M]，这个就是Part A
- 把Part A中所有 [M]位置的token提取出来并对顺序打乱，并且添加前缀 [S]，得到seq B。：[S]x5x6[S]x3
- 把Part A和Part B拼接起来，输入GLM，GLM中Part A采用双向编码器，如上右图(d)的蓝色框内。Part B是单向解码器，可以看到Part A的所有内容，以及当前时间片之前的内容，但它不能看到当前时间片之后的内容，如上图右图(d)的红绿框。


**关键点**

1. 在掩码处理中，从输入数据中随机选择一部分数据进行掩码。要服从[泊松分布](https://zhida.zhihu.com/search?content_id=240310861&content_type=Article&match_order=1&q=泊松分布&zhida_source=entity)，重复采样，直到原始tokens中有15%被mask。
2. 在GLM模型中，采用了自回归空白填充的自监督训练方式。需要设计掩码策略来生成掩码，如根据预先设定的规则来选择掩码的长度和位置。
3. 在生成掩码后，需要对掩码进行填充。在GLM模型中，采用了特殊的填充方式，如span shuffling和2D positional encoding。
4. 损失函数：在掩码处理过程中，需要根据损失函数来计算掩码处理的效果。在GLM模型中，采用了交叉熵损失函数来衡量模型在掩码处理任务上的表现。这个过程涉及到优化理论和数值分析的知识。
5. 自回归环节有位置编码环节，将位置编码的序列分为两个组，这两个组分别是：
   1. **Part A编码**：1、mask后的序列按照位置生成位置编码信息；2、采样后的序列在原始序列中的位置编码信息
   2. **Part B编码：**1、将采样多个子序列随机排列，然后每个子序列各自进行排序，互不干扰；因为输入序列是经过[mask]的原始序列+采样生成的序列，为了保持编码序列一致性，在这里原始序列的位置编码全部为0

#### Decoder-Only 架构

PT（Generative Pre-trained Transformer）是一种基于 Transformer Decoder-only 架构的自回归语言模型。GPT 在预训练阶段采用自回归任务，即根据已有的前文预测下一个 token。为了避免模型看到未来信息，GPT 使用带因果 Mask 的 Self-Attention（Masked Self-Attention），使每个位置只能关注当前位置及之前的 token。通过多层 Transformer Decoder Block，模型逐渐学习语言中的语法、语义以及世界知识，最终具备文本生成能力。

>* <strong>结构：</strong> 由多个Transformer Decoder层堆叠而成，但移除了其中的Encoder-Decoder交叉注意力部分。
>* <strong>核心机制：</strong> <strong>单向（因果）自注意力机制 (Causal Self-Attention)</strong>。在预测第 `t` 个词元时，模型只能关注到位置 `1` 到 `t-1` 的词元，不能看到未来的信息。这种自回归的特性天然适合生成任务。
>* <strong>最擅长的任务类型：自然语言生成 (NLG)</strong>。
>  * <strong>具体任务：</strong>
>    * <strong>开放式文本生成：</strong> 写文章、故事、诗歌。
>    * <strong>对话系统/聊天机器人：</strong> 如ChatGPT。
>    * <strong>代码生成：</strong> 如Copilot。
>    * <strong>上下文续写 (In-context Learning)。</strong>
>  * <strong>原因：</strong> 语言的生成过程是顺序的、从左到右的，Decoder-Only架构的单向注意力完美地模拟了这一过程。目前绝大多数的通用大语言模型都采用此架构。
>



## vLLM不知如何开始？看这篇：vLLM框架快速入门引导

[vLLM不知如何开始？看这篇：vLLM框架快速入门引导](https://zhuanlan.zhihu.com/p/1984742841528902530)

作为vLLM源码的开发者、框架的使用者，刚接触vLLM框架时会有这样的问题“**如何快速地了解vLLM全貌？**”。要解决这个问题，可以去读vLLM官方指导手册[[1\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_1)、阅读其github源码[[2\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_2)或者相关博客[[3\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_3)。然而，对于初学者这么做虽然有效，但不一定高效。推荐的一个学习步骤：

先大致了解**整体运行流程**，接着理解**关键模块逻辑**，然后学习**关键特性**。同时，逐步理解**源码**。

本文作为这个过程的一个前置引导，主要分析vLLM框架的运行流程。由于vLLM框架的迭代速度非常之快，如果直接解读源码，可能过几个月后这些逻辑就发生了较大的变化，所以文中以概念为主，代码逻辑为辅。主要参考vLLM **0.10.0，**本人梳理了一版整体架构图，如下所示，细节后面逐步展开。

![img](LLM 篇.assets/v2-2fd218f425161bbee18327321377b327_1440w.jpg)

### 1 为什么需要一个LLM推理框架？

在有PyTorch/[TensorFlow](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=TensorFlow&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJUZW5zb3JGbG93IiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.1q5Z9PtCSiK5riQMekffDoEHKhSDNqhjD0f8qP2iJ7Y&zhida_source=entity)这些既能训练又能推理的深度学习框架后，为什么还需要构建一个推理框架？这是在学习vLLM前需要思考的一个关键问题。

以PyTorch框架为例，有了训练好的模型后，切换model的模式即可运行推理：

```python
# 关键步骤：
# 1. 设置为推理模式
model.eval()

# 2. 进行推理（不计算梯度）
with torch.no_grad():  
    output = model(input_data)
```

对于自回归的[大语言模型](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=大语言模型&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLlpKfor63oqIDmqKHlnosiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.N6DYUGhgCmVjKtn9cqEotI6qNYImAs8WLclmlxbxf9I&zhida_source=entity)（LLM）多个步骤：增加前、后token的转换处理，token的[流处理](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=流处理&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmtYHlpITnkIYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.CQB1Xj3r3RqmpSg45qee6TnT3KpZL891kaQ0fcXfH_4&zhida_source=entity)：

![img](LLM 篇.assets/v2-541617787a9ead90944a74374d6d5a16_1440w.jpg)

但这种方式会面临如下问题：

1. 如何处理来自不同用户的请求？
2. 资源利用率如何维持较高水平？
3. 如何避免[超显存](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=超显存&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLotoXmmL7lrZgiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.z3K_z6sgLXxTPliEJ4r9PF4JDKbwqF_95yVeBb11eGU&zhida_source=entity)(OOM)？

对于问题1一般的解决方式是部署[推理服务](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=推理服务&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmjqjnkIbmnI3liqEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.MrLZ3jvZYQHrkw-3U2ZnW_r5DgHGKmI9TZR2No640uw&zhida_source=entity)，通过[API server](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=API+server&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJBUEkgc2VydmVyIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.OtlcakvT2kB-SH9ZXdi0PjFuVwSZckxWXA5GKr7vP-A&zhida_source=entity)来响应用户的并发请求。如[Tensorflow](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Tensorflow&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJUZW5zb3JmbG93IiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.mld2-ZQtGADH8y0wHWXF5y-4MY4yqBSqK7Vcm5guKHI&zhida_source=entity)的推理部署服务、NVIDIA的[Triton](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Triton&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJUcml0b24iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Vu_VebjrTHFGOloJ2c8IfBDbuG6f9tKzDYD2x4p-Cdk&zhida_source=entity)。

问题2、问题3主要跟负载的动态变化相关：

- 单位时间内推理服务处理的请求数量会发生动态变化，这是推理服务所共有的问题。
- 单个请求的资源需求会发生变化。这是[自回归模型](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=自回归模型&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLoh6rlm57lvZLmqKHlnosiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.gPck_rvoFOovcAwHnmLyh4GOOhD97oCE_QwCQFlnpBQ&zhida_source=entity)所特有的。 随着序列的增长，计算量、KV cache显存量均会增长。

若按照峰值需求来配置资源，会导致资源整体利用率低；若资源给得太少，会使得推理服务质量下降，或者触发OOM问题。所以针对LLM的特点，需要有一个专门[推理引擎](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=推理引擎&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmjqjnkIblvJXmk44iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.F4nQRD39iL1W_bbWXAMz-bm0C_TviIqkeEyQUbxys_o&zhida_source=entity)完成高效的**请求调度与资源分配**。

### 2 vLLM的基本要素

为了解决通用深度学习框架中存在的不足，vLLM设计了几个关键模块：

1. **调度器(**Scheduler)，用于解决多请求之间的调度协同问题；
2. **[显存管理](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=显存管理&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmmL7lrZjnrqHnkIYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Eq-MEIRHawrnpGLqK_BiyJ659sPWgiOWOmz8FlCW_qw&zhida_source=entity)(**KV cache manager)，为请求分配KV cache内存资源。
3. **执行器**(Model runner)，完成模型的计算。

上述3个模块放置在引擎核(Engine core)中。有了关键模块后，再采用API服务的方式，得到如下所示的改进方案：

![img](LLM 篇.assets/v2-1b96921220c80e061c475634e10104ea_1440w.jpg)

在这个基础上为了提升算子的下发速度，框架要做进一步优化。把在CPU侧执行的一些步骤放入独立的进程中，如请求的前置处理，token转换等，进程之间采用[异步流程](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=异步流程&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLlvILmraXmtYHnqIsiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.buHsrdxZg3KKsRC4jNiCLjLQ84dSc5vYAsKZDGYpmT8&zhida_source=entity)。这么做能够降低CPU运算对[GPU](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=GPU&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJHUFUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.EQprUQM5NOpiEbVeJsIxYGLW5_zq_P9tCwg44JDaxXM&zhida_source=entity)运算的阻塞影响。

![img](LLM 篇.assets/v2-240f596c520af982ddb7069095c8475f_1440w.jpg)

同时为了适配**分布式并行推理**，在engine core里面抽出了一个engine core client模块。该模块负责给不同的engine core分配请求。在模块管理上面，除了engine core，其它模块基本放入AsyncLLM中。这也是为了防止CPU成为运算的瓶颈。

vLLM的基本流程执行框图如下，其中Node 0为主节点，上面运行了[API Server](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=API+Server&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJBUEkgU2VydmVyIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.3lTSDDSRdMp2U3KZ_suKO9AbCSyJVAUSorEj9tZt42I&zhida_source=entity)、AsyncLLM、engine core；从节点(Node1~N)，**仅运行**engine core。

![img](LLM 篇.assets/v2-2627b373ac28f3a22c7094ed11600f2d_1440w.jpg)

按照[OpenAI](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=OpenAI&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJPcGVuQUkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.aCPOYRx9nDLn0LwcqLEpg0e6z8D8zW6K25vr4evlXK8&zhida_source=entity)的格式，给vLLM发送一个在线服务请求：

![img](LLM 篇.assets/v2-bb2ed018abbda35ed33176583d2b70ee_1440w.jpg)

收到请求后，其基本的处理步骤：

1. API server响应请求，对请求的信息进行初步处理；
2. 进行请求的前置处理，包括对prompt进行token转换获得token id；
3. engine core client将请求发送给合适engine core，engine core完成自回归运算；
4. 进行后置处理，包括token转文本的过程；
5. 返回请求结果给用户。

### 3 关键模块运行逻辑

#### 3.1 Engine core模块

单engine core的基本架构图如下所示。AsyncLLM与engine core运行在不同的进程中，两者通过队列(queue)交互。engine core的任务由[executor](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=executor&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJleGVjdXRvciIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI2Nzc5OTA1OCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.AmYmlXzr6DYuoERcj3HBcw7HDvZLKqkEBYQCSLnbX1s&zhida_source=entity)下发，多个worker共同完成LLM的数据生成。一般情况下，每个worker拥有一张GPU卡，多worker可实现TP/SP/EP等并行策略。

![img](LLM 篇.assets/v2-875294cb4d3293430a24c4e4efeefc4d_1440w.jpg)

注：上图为V1版本，与V0版本有差异，具体参看"[vLLM框架V1演进分析](https://zhuanlan.zhihu.com/p/1894423873145004335)"[[4\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_4)。

Engine core里有三个协同的线程：

1. **输入处理线程**(process input)：主要负责接收engine core client传递过来的数据，并将请求放入队列；
2. **输出处理线程**(process output)：将数据通过zmq通信返回给engine core client；
3. **处理循环线程**(**run busy loop**)：持续地从输入队列拿取请求，按照**vLLM config**里的参数构造数据，并执行引擎的step操作。step操作包括**请求调度**和**模型运算**。

![img](LLM 篇.assets/v2-d6a836de5d852cbd343b0e68650b23af_1440w.jpg)

engine core计算流

#### 3.2 Scheduler模块

scheduler模块主要职责是：根据**系统资源**和当前**在执行请求**的情况，组织每次推理需要计算的数据。

分析scheduler的执行逻辑前，先回顾下LLM推理过程的一般特点：

- 推理的阶段不一样，计算、访存的资源消耗量不同。prefill属于[计算密集](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=计算密集&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLorqHnrpflr4bpm4YiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.uOAtFGCNqgEdXIV1SxAmVOgWhFBp-GQItS8EFY21k9w&zhida_source=entity)、decode属于[访存密集](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=访存密集&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLorr_lrZjlr4bpm4YiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.KUKfvyN0qEii8my1ZHmTu1rZKZ7OHsKxwmU0Kez-5Kw&zhida_source=entity)。
- 不同请求的decode结束时机不同，生成的序列长短也不一样。

![img](LLM 篇.assets/v2-c9c366b22c4bbb1c16ae8cae937d4eb7_1440w.jpg)

批处理示意：黄色表示prefill、蓝色表示decode生成

结合LLM的推理特点，[scheduler](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=3&q=scheduler&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJzY2hlZHVsZXIiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MywiemRfdG9rZW4iOm51bGx9.iywyUVr-xqK5592Uiz_3dBiddehkkKYcaxNFUeKzayA&zhida_source=entity)目前应用了两个关键技术：

1 **[持续批处理](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=持续批处理&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmjIHnu63mibnlpITnkIYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.DQhDrkMcVxJGJzN4VuvutlVswFQAHS8aViz-wBkuQDA&zhida_source=entity)**(**Continuous-batching**)：持续地往GPU中送入请求数据，而不是离散的进行数据推理。一个请求结束立刻下发新的请求。

![动图封面](LLM 篇.assets/v2-320c5d282e26e8fd18262c29d2adb3a7_b.jpg)



continuous batching

2 **分块预填充(Chunked Prefill)**：将大的prefill分块成更小的块（切分序列）执行，也可以将它们与decode阶段的请求一起混合执行。

![img](LLM 篇.assets/v2-c86ef2055b096ac5ee8c0ae0b2cb3358_1440w.jpg)

除此之外，scheduler还要根据请求优先级调整执行顺序，高优先级的请求可以打断低优先级的请求。

**scheduler整体逻辑**：按照可用资源的数量和优先级构建调度输出。scheduler里面有两个主要的队列waiting和running，以及一些辅助队列。运行时，请求在不同的队列之间轮转。scheduler通过KV manager为请求配备KV cache。scheduler的优先级默认是[FCFS](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=FCFS&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJGQ0ZTIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.IeJhUkiXJqBGretRV0sanK0NWG85xjRB6y8-JotpeBY&zhida_source=entity)(先到先服务)，也支持用户自定义。

![img](LLM 篇.assets/v2-f17fb7e55115c2ae7fcb8ed026173058_1440w.jpg)

scheduler处理的大致步骤：

1. 请求抵达后，先进入waiting队列；
2. 找KV manager申请KV cache块；
3. 具备下发条件的请求转入running队列，组batch下发执行(图示中有3个请求)；资源不足的请求会转回[waiting队列](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=2&q=waiting队列&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJ3YWl0aW5n6Zif5YiXIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjIsInpkX3Rva2VuIjpudWxsfQ.6k_ndXXQfVgQXGE7PsWMEpGlYmZBwjc6fAbCZK0NZNs&zhida_source=entity)。

更细节的执行步骤参考[[5\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_5)：

![img](LLM 篇.assets/v2-41db92a2cf13be6695a6eecdcb63b8e0_1440w.jpg)

如果觉得Scheduler逻辑太复杂，理解起来费劲，可以先阅读：

[![img](LLM 篇.assets/v2-9ccd7b7226fe4105631ff4304784d4f7.gif)vLLM Scheduler逻辑难啃？先手搓一个基础调度器244 赞同 · 11 评论 ](https://zhuanlan.zhihu.com/p/1988193790129902960)文章

#### 3.3 KV Manager模块

[KV值](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=KV值&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJLVuWAvCIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI2Nzc5OTA1OCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.KdFqgHdAkR6yagzW0Gcmx_oaUNri6AatJG2dNpQazJY&zhida_source=entity)的复用能够降低 Attention中的冗余计算，目前KV cache已成为了Attention[推理计算](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=推理计算&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmjqjnkIborqHnrpciLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.3r6coQJBlBE30NLtLbZGsocTwfIdTKHxYsTvsbltVlQ&zhida_source=entity)的标准配置。框架中需要完成对多个不同请求的KV cache管理。

![动图封面](LLM 篇.assets/v2-17a8bb267df5a7afdb0a261cdec7f4b6_b.jpg)



vLLM中的KV cache管理逻辑基于**Paged attention**原理，目前的版本还融合了**[前缀树](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=前缀树&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLliY3nvIDmoJEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.UUxKLvViEXwfWcXfFIsrvCVJW0NoB7wC1TE1t0lQtz8&zhida_source=entity)**特点[[6\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_6)。

![动图封面](LLM 篇.assets/v2-eefbaead8f4d070e2b5c3834a1ed69df_b.jpg)



KV cache管理的整体架构示意图如下所示，分为了[逻辑层](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=逻辑层&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLpgLvovpHlsYIiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.cM1uFpkhYwaUCyMSK1Nu4SVWSHloPu5YOfSSVCaI3Hg&zhida_source=entity)和物理层。KV Manager负责逻辑层、Model Runner处理物理层；Scheduler（调度器）作为信息传递的桥梁，衔接了逻辑层与物理层。cache的管理元素包括：池（pool）表(table)、层(layer)、块(block)和槽(slot)。

- **slot：**为最小[管理单元](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=管理单元&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLnrqHnkIbljZXlhYMiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.qngbZ-ie147Hd66Pau1mbR4YQ-9CoyyMvN1aVGZAnXg&zhida_source=entity)，每个token占一个slot；
- **block：**为请求分配的基本单位，一个block包含多个slot；
- **pool：**为逻辑层block的管理合集，通过[链表](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=链表&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLpk77ooagiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.E4ctvIKr0Z-Zku0xWobMDa9Ve4fUSrYD2fLgRf4C9mM&zhida_source=entity)将block数据组织起来；
- **table：**管理请求与数据的[映射表](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=映射表&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLmmKDlsITooagiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.8ib8r6oDLdum9PW7PojjEuAEMQECPaqp1I9gTZ9Pg24&zhida_source=entity)，一个table可包含多个请求的信息。位于物理层；
- **layer：**一个整体的[tensor](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=tensor&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJ0ZW5zb3IiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.zwvyp2uz87wJ6FAARwDxRVAWFT_uw39lzCJ1uusdJeI&zhida_source=entity)，拆分成多个blocks使用。对应attention的一个层，所有请求共用；

![img](LLM 篇.assets/v2-d7d7975afcef0d34eee6d5f4973cb6e3_1440w.jpg)

模块之间运行的关键步骤：

1. Scheduler分配资源给请求，通过KV Manager申请逻辑blocks；
2. KV Manager把Pool中空闲的blocks选中后给到对应请求；
3. 分配好逻辑blocks后Scheduler构建scheduler.output传递给ModelRunner；
4. ModelRunner为每条请求创建block table，并生成slot_mapping；
5. 计算时把[slot_mapping](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=2&q=slot_mapping&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJzbG90X21hcHBpbmciLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MiwiemRfdG9rZW4iOm51bGx9.ORqX1SO0fTx-Tv_NvU-8mlXlHKsX842IwehcMiOFyyQ&zhida_source=entity)传入attention，就能够从物理KV blocks上面找到所需的数据。

逻辑KV blocks是一个[双向链表](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=双向链表&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiLlj4zlkJHpk77ooagiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.jiGUhxH2dHOroMaYNOVVPzXM2ZFTBJgkgQR6Ob0oBwI&zhida_source=entity)，采用LRU策略淘汰旧数据[[7\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_7)。

![img](LLM 篇.assets/v2-8a9dcac89035c174e49c5f0bef7afb4f_1440w.jpg)

blocks数据结构示意

KV Manager的代码解析参看“[vLLM V1 KV cache 管理机制剖析](https://zhuanlan.zhihu.com/p/1954128446398633139)”[[8\]](https://zhuanlan.zhihu.com/p/1984742841528902530#ref_8)

#### **3.4 Model Runner**

模型执行器(model runner)主要负责计算调度器发送过来的批请求，并返回执行结果。

从上面engine core的架构可知，executor可以有多个worker模块，每个worker都会有自己的model runner。model runner的逻辑主要是模型运算、以及物理层的kv cache分配与管理。

![img](LLM 篇.assets/v2-cf19c05f755724e2762479d017e276a6_1440w.jpg)

执行的基本步骤：

1. 根据映射表(block table)信息为每个待执行请求分配kv blocks；
2. 将请求组成序列batch，并让模型处理该batch数据。
3. 在Attention层运算阶段，每层拿取自己对应的kv cache数据，完成MHA/GQA/MLA运算。

#### 3.5 Attention模块

Attetnion模块负责承载注意力计算的算子，其关键要素：

- **QKV数据**：[Q值](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Q值&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJR5YC8IiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.ZxhZa4QRaV2VzkXJTATllzzMv0XQ_a6XkPnFEgVqAd0&zhida_source=entity)是展平后的tokens序列，KV则是整个KV [Cache](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Cache&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJDYWNoZSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI2Nzc5OTA1OCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.7wSlHno8VHc70IrRpP7UeRGH4RZ1CPh_8VpNeM5cfKg&zhida_source=entity) Tensor。
- **[Metadata](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Metadata&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJNZXRhZGF0YSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI2Nzc5OTA1OCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.xendXX03fyH07kIU7URN30fLzrV5xSxZYTpm1G9sq-E&zhida_source=entity)**：注意力运算的元数据，包括KV相关的block table、slot_mapping，以及Q值的起始位置信息(query_start_loc)，用于区分不同请求；
- **[Backend](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Backend&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJCYWNrZW5kIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjY3Nzk5MDU4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.woDVb3DL5Macg62wzOEn8XhtGOxxjThCVvXwVyGx4O0&zhida_source=entity)**：通过定义不同后端(backend)来支持不同类型的[Attention算子](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=Attention算子&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJBdHRlbnRpb27nrpflrZAiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.ytHOSgtzhNIIis_EjH795DxGH5Kqts_cdXchKsQZXtU&zhida_source=entity)，以及不同的硬件。

![img](LLM 篇.assets/v2-6181fc87df37a241b706e1476d88b80e_1440w.jpg)

### 4 数据处理流程

接下来通过一个请求处理的数据流的例子，了解从请求、[kv cache](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=3&q=kv+cache&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJrdiBjYWNoZSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI2Nzc5OTA1OCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjozLCJ6ZF90b2tlbiI6bnVsbH0.1BTjysz7uQ-HTglH739rQ4qviF30olqaLY4IjdVTqok&zhida_source=entity)、[attention算子](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=attention算子&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJhdHRlbnRpb27nrpflrZAiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.tS3JAA-dVE_iwqvJeU7I7RjSsamvidDNDNwbr5JM56Q&zhida_source=entity)、到采样的数据传递过程。

给vLLM发送请求，提示词prompts包含两条语句，即有两个子请求。

**方式一**：在线服务(online serving)：

![img](LLM 篇.assets/v2-81c9aceb24440d2dd54b58bce9905f7b_1440w.jpg)

**方式二**：离线推理(offline infer):

![img](LLM 篇.assets/v2-63dd3bce01331c1cbac293fe51935e8d_1440w.jpg)

**Step1**：**文字转token ids**(Tokenization)。 多个请求会拼接成一个数据(组batch)，用位置(positions)记录每个请求对应的ids。

![img](LLM 篇.assets/v2-ce64a92e12e39fb37f88a52183e80c32_1440w.jpg)

**Step2：**KV manager分配逻辑块、计算slot。

![img](LLM 篇.assets/v2-de817cf81d97a03250793e73cd676d0a_1440w.jpg)

**Step3：** Model runner分配KV cache

![img](LLM 篇.assets/v2-6e30fb8051716927d32b368428cdb08a_1440w.jpg)

**Step4：** Decoding生成新token，并更新ids、positions、slot_mapping数据。该过程需要迭代多次。

![img](LLM 篇.assets/v2-575bc6074232352cb6aeb9ec5a4ccdd1_1440w.jpg)

示例中KV manager的逻辑块是连续的，而物理块在model runner中不连续。

从模型输出的[logits](https://zhida.zhihu.com/search?content_id=267799058&content_type=Article&match_order=1&q=logits&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODU1OTM0NTYsInEiOiJsb2dpdHMiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjc3OTkwNTgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.bVMoEN2-UOn9eQmi7DZ0bQ5N9nfgqF4RDeZDQ6NaROQ&zhida_source=entity)到token id，要经过**采样(sampling)**计算。

![img](LLM 篇.assets/v2-62c8e41aa612c81a6317239216470732_1440w.jpg)

**Step5：**将token id还原成词(De-Tokenization)。

![img](LLM 篇.assets/v2-5c33090efacfb534346fdde8554c8c44_1440w.jpg)

### 5 概念与关键特性

- **关键特性与相关术语**（如TTFT/TPOT）**推荐阅读**：

[![img](LLM 篇.assets/v2-932106b42a2188d44bbbcdc286217515.jpg)大模型推理核心概念与术语总结274 赞同 · 23 评论 ](https://zhuanlan.zhihu.com/p/1983137653336585901)文章

- **投机推理(speculative decoding)：**

[![img](LLM 篇.assets/v2-71a0970082de04780a2bca6ff020e19d.jpg)LLM提速利器：投机推理的原理与常见方案90 赞同 · 10 评论 ](https://zhuanlan.zhihu.com/p/1978037808544370747)文章

- **推理采样**：

[![img](LLM 篇.assets/v2-9ff9520a019a48851c2bded8566c61ff.jpg)LLM推理采样(Sampling)常见知识概览87 赞同 · 9 评论 ](https://zhuanlan.zhihu.com/p/1981752176578667658)文章

- **Multi-Lora服务：**

[![img](LLM 篇.assets/v2-aa59122fe4d9bdf2694a54ae4990ffc3.jpg)从LoRA到Multi-LoRA：原理&代码实践73 赞同 · 4 评论 ](https://zhuanlan.zhihu.com/p/1984729458444363168)文章

- **自动前缀匹配(APC):**

[![img](LLM 篇.assets/v2-52534dc942694d45fb0e1d08086d8f31.jpg)vLLM的prefix cache为何零开销366 赞同 · 34 评论 ](https://zhuanlan.zhihu.com/p/1896927732027335111)文章

- **Chunked Prefill&FLashDecoding**：

[![img](LLM 篇.assets/v2-1c689ede21ed5879adcc17c00ae8f777.jpg)推理长序列利器：ChunkedPrefill&FlashDecoding原理详解109 赞同 · 18 评论 ](https://zhuanlan.zhihu.com/p/1988996116017086993)文章

- **并行优化基础：**

[![img](LLM 篇.assets/v2-7a148edaef14d047dd0b0f4a694ad786.jpg)LLM推理并行优化的必备知识373 赞同 · 14 评论 ](https://zhuanlan.zhihu.com/p/1937449564509545940)文章

------

**vLLM的框架其它内容参看：**

[![img](LLM 篇.assets/v2-f9e3e6f8051a9edc65c5df5de61701f3.jpg)LLM推理知识指南---kaiyuan207 赞同 · 5 评论 ](https://zhuanlan.zhihu.com/p/1954137524881580796)文章

> 想深耕AI Infra领域？欢迎访问**[InfraTech](https://link.zhihu.com/?target=https%3A//github.com/CalvinXKY/InfraTech)**库！内容涵盖大模型基础、PyTorch/vLLM/SGLang框架入门、性能加速等核心方向，配套**[50+知识干货](https://link.zhihu.com/?target=https%3A//github.com/CalvinXKY/InfraTech)**及适合初学者的**[notebook](https://link.zhihu.com/?target=https%3A//github.com/CalvinXKY/InfraTech)**练习。



## 请你详细介绍ROPE，对比绝对位置编码它的优劣势分别是什么？

**RoPE (Rotary Position Embedding) 介绍**
RoPE，全称旋转位置编码，是目前大语言模型（如Llama系列、Qwen等）中最主流的位置编码方案之一。它是一种将位置信息融入自注意力机制的创新方法。

其核心思想是：<strong>通过向量旋转的方式，将绝对位置信息编码到Query和Key向量中，从而使得模型在计算注意力分数时，能够自然地利用相对位置信息。</strong>

<strong>工作原理：</strong>
RoPE不再像传统位置编码那样直接将位置向量加到词嵌入上。它的操作发生在生成Q和K向量之后、计算注意力分数之前：

1.  <strong>维度分组：</strong> 将Q和K向量的 $d$ 维特征两两一组，视为 $d/2$ 个二维向量。
2.  <strong>构造旋转矩阵：</strong> 对于序列中的位置 $m$，构造一个与位置相关的旋转矩阵 $R_m$。这个矩阵在二维空间中表示一个旋转操作。
3.  <strong>旋转Q和K：</strong> 将每个二维向量组通过对应的旋转矩阵 $R_m$ 进行旋转。

数学上，这个过程等价于将每个二维向量 $(x_m, x_{m+1})$ 看作一个复数，然后乘以一个复数 $e^{im\theta}$，其中 $m$ 是位置， $\theta$ 是一个预设的、与维度相关的常数。这个操作只会改变向量的相位（方向），而不改变其模（长度）。

<strong>关键特性：</strong>
RoPE的巧妙之处在于，经过旋转后的两个位置 $m$ 和 $n$ 的Query向量 $q_m$ 和Key向量 $k_n$ 进行点积运算时，其结果只与它们的相对位置 $(m-n)$ 有关，而与它们的绝对位置 $m$ 和 $n$ 无关。这使得自注意力机制天然地具备了对相对位置的感知能力。

<strong>对比绝对位置编码的优劣势：</strong>

<strong>RoPE的优势：</strong>

1.  <strong>内置相对位置建模：</strong> 这是其最大的优势。RoPE使得注意力分数直接依赖于词元间的相对距离，这更符合自然语言中语法和语义依赖通常是相对的这一特性。
2.  <strong>良好的外推能力：</strong> 由于其数学性质，RoPE在处理比训练时更长的序列时表现出色，具有很强的长度泛化能力，这也是长序列LLM偏爱它的重要原因。
3.  <strong>不引入额外可训练参数：</strong> RoPE是一种函数式的、固定的编码方式，不需要像可学习位置编码那样占用模型参数。
4.  <strong>随着距离增加，依赖性衰减：</strong> 旋转的性质使得距离越远的词元，其内积关系会呈现周期性的衰减，符合语言中距离越远相关性越弱的直觉。

<strong>RoPE的劣势：</strong>

1.  <strong>理论理解相对复杂：</strong> 其背后的数学原理（复数、欧拉公式、旋转矩阵）比直接相加的绝对位置编码更抽象。
2.  <strong>对绝对位置信息的表征可能较弱：</strong> 虽然RoPE从绝对位置导出，但其在注意力机制中的核心作用是体现相对位置。对于那些强依赖绝对位置信息的特定任务（例如，判断一个词是否在句子开头），它的效果可能不如直接使用绝对位置编码直观。

![48947df6-6c70-4899-9ef1-30509688c820](LLM 篇.assets/48947df6-6c70-4899-9ef1-30509688c820.png)



## 你知道MHA，MQA，GQA的区别吗？

MHA、MQA和GQA是Transformer模型中三种不同的注意力机制变体，它们的主要区别在于如何组织和共享Query、Key和Value的“头”（Head），核心目标是在模型效果和推理效率（特别是显存占用）之间做出不同的权衡。

#### 1. MHA (Multi-Head Attention)

这是原始Transformer论文中提出的标准注意力机制。

* <strong>工作原理：</strong>
  1.  将输入的Q、K、V向量分别通过 $N$ 个独立的线性变换，得到 $N$ 组不同的 $Q_i, K_i, V_i$ 头（ $i=1, ..., N$ ）。
  2.  这 $N$ 组头在各自的子空间中并行地计算注意力（Scaled Dot-Product Attention）。
  3.  将 $N$ 个头计算得到的输出向量拼接（Concatenate）起来。
  4.  最后通过一个线性变换将拼接后的向量映射回原始维度。
* <strong>结构：</strong> $N$ 个Query头， $N$ 个Key头， $N$ 个Value头。
* <strong>优点：</strong> 效果最好，模型能力最强。每个头可以在不同的表示子空间中学习到不同的信息。
* <strong>缺点：</strong> 推理成本高。在自回归生成任务中，需要缓存每一层的Key和Value（即KV Cache），MHA的KV Cache大小与头的数量$N$成正比，显存占用非常大，限制了长序列的生成。

#### 2. MQA (Multi-Query Attention)

为了解决MHA在推理时的显存瓶颈而被提出。

* <strong>工作原理：</strong>
  1.  与MHA一样，有 $N$ 个独立的Query头。
  2.  <strong>核心区别：</strong> 所有的 $N$ 个Query头共享<strong>同一个</strong>Key头和<strong>同一个</strong>Value头。
* <strong>结构：</strong> $N$ 个Query头，<strong>1个</strong>Key头，<strong>1个</strong>Value头。
* <strong>优点：</strong> 极大地降低了推理成本。KV Cache的大小不再依赖于头的数量 $N$ ，相比MHA减小了 $N$ 倍，显著降低了显存占用，并加快了推理速度。
* <strong>缺点：</strong> 可能会导致模型性能的下降。因为所有Query头被迫从同样的一组Key和Value中提取信息，模型的表达能力受到了一定的限制。

#### 3. GQA (Grouped-Query Attention)

GQA是MHA和MQA之间的一个折中方案，旨在平衡性能和效率。

* <strong>工作原理：</strong>
  1.  将 $N$ 个Query头分成 $G$ 组。
  2.  <strong>核心区别：</strong> 每组内的Query头共享一个Key头和一个Value头。总共有 $G$ 个Key头和 $G$ 个Value头。
* <strong>结构：</strong> $N$ 个Query头，<strong>G个</strong>Key头，<strong>G个</strong>Value头。（通常 $1 < G < N$ ）。
* <strong>说明：</strong>
  * 当 $G=N$ 时，GQA等价于MHA。
  * 当 $G=1$ 时，GQA等价于MQA。
* <strong>优点：</strong> 在推理效率上远超MHA，同时在模型性能上优于MQA。它提供了一个灵活的旋钮，可以根据具体需求在效率和效果之间进行调整。Llama 2等模型就采用了GQA。

<strong>总结：</strong>

| 特性                      | MHA (Multi-Head Attention) | MQA (Multi-Query Attention) | GQA (Grouped-Query Attention) |
| :------------------------ | :------------------------- | :-------------------------- | :---------------------------- |
| <strong>结构</strong>     | N个Q头, N个K头, N个V头     | N个Q头, 1个K头, 1个V头      | N个Q头, G个K头, G个V头        |
| <strong>模型质量</strong> | 最高                       | 可能下降                    | 接近MHA，优于MQA              |
| <strong>推理效率</strong> | 最低 (KV Cache大)          | 最高 (KV Cache小)           | 居中，远好于MHA               |
| <strong>应用</strong>     | BERT, GPT-3                | PaLM                        | Llama 2, Mixtral              |



## 什么是Scaling Laws？它揭示了模型性能、计算量和数据量之间的什么关系？

**什么是Scaling Laws？**
Scaling Laws（尺度定律）是由OpenAI、DeepMind等机构通过大量实验发现的一系列经验性规律。它揭示了大型语言模型的性能（通常以交叉熵损失函数Loss来衡量）与三个关键资源要素——<strong>模型参数规模（N）</strong>、<strong>训练数据集大小（D）</strong>和<strong>训练所用的计算量（C）</strong>——之间存在着可预测的<strong>幂律关系（Power-Law Relationship）</strong>。

**揭示了什么关系？**

1.  <strong>性能的可预测性：</strong> Scaling Laws表明，模型的性能损失会随着N、D、C的增加而平滑地、可预测地下降。这种关系可以用一个幂律公式来描述，例如，当数据和计算量足够时，模型损失 L 与模型参数量 N 的关系大致为： $L(N) \propto N^{-\alpha}$ ，其中 $\alpha$ 是一个小的正指数。这意味着我们可以通过在小规模模型上的实验结果，来外推（predict）更大规模模型可能达到的性能。
2.  <strong>瓶颈效应：</strong> 模型的最终性能会被N、D、C中最受限的那个因素所制约。如果仅仅增加模型大小而不增加数据量，性能提升会很快达到瓶颈；反之亦然。为了有效提升模型性能，必须协同扩展这三个要素。
3.  <strong>资源的最优分配：</strong> 对于一个给定的计算预算（FLOPs），存在一个最优的模型大小（N）和数据量（D）的组合。DeepMind的Chinchilla论文是一个里程碑式的发现，它修正了早期认为应该优先扩大模型规模的观点，指出<strong>为了达到计算最优，模型参数量和训练数据量应该近似1:20的比例进行扩展</strong>。例如，训练一个70B参数的模型，大约需要1.4万亿个token的数据。

<strong>对LLM研发的指导意义：</strong>

1.  <strong>科学指导项目规划：</strong> 在投入数百万甚至数千万美元进行一次大规模训练之前，研究机构可以先通过小规模实验拟合出自己数据集和模型架构下的Scaling Law。这使得他们能够科学地预测最终模型的性能，评估项目的投资回报率，并合理申请计算资源。
2.  <strong>优化资源配置，避免浪费：</strong> Scaling Laws，特别是Chinchilla定律，为如何高效使用计算预算提供了明确的指导。它告诉我们，与其训练一个参数巨大但数据不足的模型（over-trained），不如用同样的算力去训练一个参数稍小但数据更充分的模型（under-trained），后者效果可能更好。这促使业界从单纯追求“大参数”转向“大参数与大数据的平衡”。
3.  <strong>强调数据的重要性：</strong> Scaling Laws的发现，让学术界和工业界都更加深刻地认识到，高质量、大规模的训练数据和模型参数规模同等重要，甚至在某些阶段更为关键。这推动了数据工程、数据清洗和高质量合成数据生成等领域的发展。



## 在LLM的推理阶段，有哪些常见的解码策略？请解释 Greedy Search, Beam Search, Top-K Sampling 和 Nucleus Sampling (Top-P) 的原理

在LLM的推理（或称解码）阶段，模型会生成一个词元概率分布，解码策略决定了如何从这个分布中选择下一个词元。常见的策略可以分为确定性和随机性两类。

#### 1. Greedy Search (贪心搜索)

* <strong>原理：</strong> 在每个时间步，总是选择当前概率分布中概率最高的那个词元作为输出。
* <strong>优点：</strong>
  * <strong>速度快：</strong> 计算开销最小，实现最简单。
* <strong>缺点：</strong>
  * <strong>局部最优：</strong> 每一步的“贪心”选择可能导致整个序列不是全局最优的。一个高概率的词后面可能跟着一系列低概率的词，最终序列的总概率反而不高。
  * <strong>缺乏多样性：</strong> 输出是完全确定的，对于同一个输入，每次生成的结果都一样，内容往往比较呆板、重复。

#### 2. Beam Search (集束搜索)

* <strong>原理：</strong> 这是对贪心搜索的改进。它在每个时间步会保留 $k$ 个（ $k$ 称为 "beam width" 或 "beam size"）最有可能的候选序列。在下一步，它会从这 $k$ 个候选序列出发，生成所有可能的下一个词元，然后从所有这些扩展出的新序列中，再次选出累计概率最高的 $k$ 个。最后，从最终的 $k$ 个完整序列中选择最优的一个。
* <strong>优点：</strong>
  * <strong>质量更高：</strong> 通过探索更广的搜索空间，通常能找到比贪心搜索概率更高、质量更好的序列。
* <strong>缺点：</strong>
  * <strong>计算成本高：</strong> 需要维护 $k$ 个候选序列，计算和内存开销是贪心搜索的 $k$ 倍。
  * <strong>仍然倾向于安全和高频：</strong> 优化目标是全局概率，这使得它还是倾向于生成常见、安全的句子，可能缺乏创造性，并且在长文本生成中容易出现重复。

#### 3. Top-K Sampling (Top-K 采样)

* <strong>原理：</strong> 这是一种随机采样策略。在每个时间步，不再是选择最优的，而是：
  1.  从整个词汇表的概率分布中，筛选出概率最高的 $K$ 个词元。
  2.  将这 $K$ 个词元的概率进行归一化（使它们的和为1）。
  3.  在这 $K$ 个词元中，根据新的概率分布进行随机采样。
* <strong>优点：</strong>
  * <strong>增加多样性：</strong> 引入了随机性，使得生成内容更加丰富、有趣和不可预测。
  * <strong>避免低概率词：</strong> 通过限制在Top-K范围内，过滤掉了那些概率极低、可能不通顺或奇怪的词元。
* <strong>缺点：</strong>
  * <strong>K值固定：</strong> $K$ 是一个固定的超参数。当概率分布很尖锐时（模型非常确定下一个词），一个大的K可能会引入不相关的词；当概率分布很平坦时（模型不确定），一个小的K可能会限制模型的选择。

#### 4. Nucleus Sampling / Top-P Sampling (核心采样)

* **原理：** 这是对Top-K采样的改进，它使用一个动态的候选词元集。
  1.  将所有词元按概率从高到低排序。
  2.  从概率最高的词元开始，逐个累加它们的概率，直到总概率之和超过一个预设的阈值 $p$（例如 $p=0.95$）。
  3.  这个累加过程中包含的所有词元构成了“核心（Nucleus）”候选集。
  4.  然后，在这个动态大小的候选集中，根据它们的原始概率进行归一化和随机采样。
* <strong>优点：</strong>
  * <strong>自适应候选集：</strong> 候选集的大小会根据上下文动态变化。当模型对下一个词非常确定时，概率分布尖锐，可能只有一两个词的概率和就超过了 $p$，候选集就很小，生成更精确；当模型不确定时，概率分布平坦，需要包含更多词才能达到 $p$，候选集就变大，允许更多探索。
  * <strong>兼顾质量与多样性：</strong> 相比Top-K，它是一种更原则性和鲁棒性的方法，是目前大多数LLM应用默认的采样策略。

## 什么是词元化？请比较一下 BPE 和 WordPiece 这两种主流的子词切分算法。

**什么是词元化（Tokenization）？**
词元化是将原始的文本字符串分解成一个个独立的单元（称为“词元”或“token”），并将这些词元映射到唯一的整数ID的过程。这是自然语言处理模型处理文本的第一步，因为模型只能处理数字输入。

现代大型语言模型普遍采用 <strong>子词（Subword）</strong> 词元化算法，它介于按词切分和按字符切分之间。这样做的好处是：

1.  <strong>有效处理未登录词（OOV）：</strong> 任何罕见词或新词都可以被拆解成已知的子词组合，避免了“未知”标记。
2.  <strong>平衡词表大小与序列长度：</strong> 相比于词级别，词表规模大大减小；相比于字符级别，生成的序列长度又不会过长，兼顾了效率。
3.  <strong>保留形态信息：</strong> 像 "running", "runner" 这样的词可以共享 "run" 这个子词，使得模型能够理解词根和词缀的关系。

<strong>BPE vs. WordPiece</strong>

BPE和WordPiece是两种最主流的子词切分算法，它们构建词表的过程相似，但在合并子词的决策标准上有所不同。

#### <strong>BPE (Byte Pair Encoding)</strong>

* <strong>工作原理：</strong>
  1.  <strong>初始化：</strong> 词汇表由语料库中出现的所有基本字符组成。
  2.  <strong>迭代合并：</strong> 重复以下步骤直到达到预设的词表大小：
      a.  在整个语料库中，统计所有相邻词元对的出现频率。
      b.  找出<strong>频率最高</strong>的那个词元对（例如 `('e', 's')`）。
      c.  将这个词元对合并成一个新的、更长的词元（`'es'`），并将其加入词汇表。
      d.  在语料库中，用新词元替换所有出现的该词元对。
* <strong>应用模型：</strong> GPT系列、Llama等。
* <strong>特点：</strong> 算法思想简单直观，完全基于数据中符号对的出现频率。

#### <strong>WordPiece</strong>

* <strong>工作原理：</strong>

  1.  <strong>初始化：</strong> 与BPE一样，词汇表也从所有基本字符开始。
  2.  <strong>迭代合并（核心区别）：</strong> WordPiece在选择合并哪两个子词时，不是基于频率，而是基于<strong>语言模型的似然（Likelihood）</strong>。它会尝试所有可能的合并，并选择那个能够<strong>最大程度提升训练数据似然值</strong>的合并操作。

  * 可以通俗地理解为：如果把语料库看作一个语言模型，每次合并都应该让这个语言模型产生当前语料库的概率变得最大。它倾向于合并那些内部凝聚力更强的字符组合。

* <strong>应用模型：</strong> BERT, DistilBERT, Electra。

* <strong>特点：</strong> WordPiece在切分时，通常会在单词的非起始部分子词前加上特殊符号（如`##`），例如 "tokenization" 可能会被切分为 `("token", "##ization")`。

<strong>主要区别总结：</strong>

| 特性                          | BPE (Byte Pair Encoding)                                  | WordPiece                                                    |
| :---------------------------- | :-------------------------------------------------------- | :----------------------------------------------------------- |
| <strong>合并决策标准</strong> | <strong>频率驱动</strong>：合并出现次数最多的相邻子词对。 | <strong>似然驱动</strong>：合并能最大化提升语料库语言模型似然的子词对。 |
| <strong>理论基础</strong>     | 数据压缩算法，简单高效。                                  | 概率语言模型，理论上更优。                                   |
| <strong>应用代表</strong>     | GPT, Llama, RoBERTa                                       | BERT, T5                                                     |



## 开源框架了解过哪些？Qwen，Deepseek的论文是否有研读过，说一下其中的创新点主要体现在哪？

**开源框架：**

* <strong>基础框架：</strong> <strong>PyTorch</strong> 是目前大模型研究和开发的事实标准，提供了灵活的张量计算和自动微分能力。
* <strong>模型与生态：</strong> <strong>Hugging Face Transformers</strong> 是最重要的模型库和生态系统，它极大地降低了使用和分享模型的门槛。
* <strong>大规模训练：</strong> <strong>DeepSpeed</strong> (微软) 和 <strong>Megatron-LM</strong> (英伟达) 是进行大规模分布式训练的核心框架，它们实现了上述的3D并行、ZeRO等关键技术。
* <strong>高效推理：</strong> <strong>vLLM</strong>, <strong>TensorRT-LLM</strong> 等框架专注于优化LLM的推理速度和吞吐量，通过PagedAttention等技术来解决KV Cache的显存瓶颈。

<strong>Qwen系列（可以参考开源论文自行回答，Qwen2.5，Qwen3系列）</strong>

<strong>Deepseek系列（可以参考开源论文自行回答，如GRPO）</strong>





## 你觉得NLP和LLM最大的区别是什么？两者有何共同和不同之处？

NLP（自然语言处理）和LLM（大型语言模型）之间是领域与技术、一般与具体的关系。LLM是NLP发展至今最前沿、最具影响力的一项技术范式，它在很大程度上重塑了NLP领域。

<strong>共同之处：</strong>

* <strong>最终目标一致：</strong> 两者的根本目标都是实现人工智能对人类语言的理解、生成、和运用，即所谓的“人工智能皇冠上的明珠”。
* <strong>技术根基相通：</strong> 现代NLP和LLM都建立在深度学习，特别是神经网络的基础上。Transformer架构是连接两者的关键桥梁，从BERT到GPT，都是其思想的延伸和发展。

<strong>最大的区别与不同之处：</strong>

最大的区别在于<strong>研究和应用范式</strong>的根本性转变，从“为每个任务训练一个模型”转向“用一个模型解决所有任务”。

具体可以从以下几个维度来看：

1.  <strong>任务处理范式 (Task-Handling Paradigm)：</strong>
    * <strong>传统NLP：</strong> 奉行“分而治之”的策略。研究者会针对每一个具体的NLP任务（如机器翻译、情感分析、命名实体识别）设计特定的模型架构、损失函数和训练数据集，遵循`Pre-train -> Fine-tune`的流程。每个模型都是一个“专家”。
    * <strong>LLM：</strong> 追求“大一统”的通用模型。通过在海量数据上进行大规模预训练，一个LLM基础模型就具备了解决多种任务的潜力。用户通过设计不同的 <strong>提示（Prompt）</strong> 或提供 <strong>上下文示例（In-context Learning）</strong> 来引导模型完成任务，大大简化了开发流程，甚至实现了 <strong>零样本（Zero-shot）</strong> 和 <strong>少样本（Few-shot）</strong> 学习。

2.  <strong>模型能力与“涌现” (Model Capabilities & Emergence)：</strong>
    * <strong>传统NLP：</strong> 模型的能
      力是明确且有限的，通常与其训练目标直接相关。
    * <strong>LLM：</strong> 当模型规模（参数、数据、算力）跨越某个阈值后，会表现出小模型上不存在的 <strong>“涌现能力” (Emergent Abilities)</strong> 。例如，复杂的逻辑推理（思维链, Chain-of-Thought）、代码生成、遵循复杂指令等。这些能力不是被直接训练的，而是从海量数据中自发学习到的。

3.  <strong>规模 (Scale)：</strong>
    * <strong>传统NLP：</strong> 模型参数量通常在百万级到几亿级（例如，BERT-base约1.1亿）。
    * <strong>LLM：</strong> 参数量从百亿（Billion）起步，发展到千亿甚至万亿级别。训练数据和所需计算资源也比传统NLP模型高出几个数量级。

4.  <strong>交互与应用方式 (Interaction & Application)：</strong>
    * <strong>传统NLP：</strong> 通常以API形式被集成到软件中，输入输出格式相对固定。
    * <strong>LLM：</strong> 催生了以<strong>对话</strong>和<strong>指令</strong>为核心的全新交互方式（如ChatGPT），使得AI更加平易近人。应用也从后端工具演变为可以直接面向用户的产品。

<strong>总结：</strong> 如果说传统NLP是在打造一支由各种“工具专家”组成的工具箱，那么LLM则是在努力打造一个“瑞士军刀”式的通用智能工具，它可能在某些特定任务上不如专用工具精细，但其通用性、灵活性和强大的涌现能力是前所未有的。



## “涌现能力”是大型模型中一个备受关注的现象，请问你如何理解这个概念？它通常在模型规模达到什么程度时出现？

**对“涌现能力”的理解：**
“涌现能力”（Emergent Abilities）是指那些<strong>在小型模型中不存在或表现不佳，但当模型规模（包括参数量、训练数据和计算量）达到某个临界点后，突然出现并显著超越随机水平的能力</strong>。

它的核心特征是<strong>非线性和不可预测性</strong>：

* <strong>非线性增长：</strong> 这种能力的性能表现并不随着模型规模的增加而平滑、线性地提升。相反，它会在某个规模区间内发生“相变”式的跃迁，性能从接近随机猜测的水平迅速提升到非常高的水平。
* <strong>非直接训练：</strong> 这些高级能力通常不是通过特定的监督学习目标直接训练出来的。例如，我们没有直接教模型如何“一步一步思考”，但当模型足够大时，它通过学习海量文本中的逻辑关系，自发地获得了这种能力。

<strong>典型的涌现能力例子包括：</strong>

1.  <strong>思维链（Chain-of-Thought, CoT）：</strong> 在面对需要多步推理的数学或逻辑问题时，通过提示模型“一步一步地思考”，大模型可以生成一个连贯的推理过程并得出正确答案。小模型则无法利用这种提示。
2.  <strong>上下文学习（In-context Learning）：</strong> 无需更新模型权重，仅在Prompt中提供几个任务示例（Few-shot），大模型就能“学会”并执行这个新任务。
3.  <strong>执行复杂指令：</strong> 理解并遵循包含多个步骤、约束和否定逻辑的复杂人类指令。

<strong>出现的模型规模：</strong>
涌现能力出现的具体规模<strong>没有一个固定的数值</strong>，它取决于能力本身、模型架构、数据质量和评估任务的复杂性。

然而，根据Google等机构的标志性研究，许多引人注目的涌现能力，例如<strong>思维链推理</strong>，通常是在模型参数规模达到<strong>百亿（tens of billions）到千亿（a hundred billion）</strong> 级别时开始出现的。

* 例如，在Google PaLM模型的实验中，思维链推理能力在<strong>62B参数</strong>的模型上开始显现，而在8B和16B的模型上则完全无效。这种能力随着模型增长到<strong>540B</strong>时变得更加强大和稳定。

总而言之，“涌现能力”是“量变引起质变”在大型模型领域的生动体现，它表明单纯地扩大规模可以解锁全新的、更高级的认知能力，这也是当前LLM研究持续推动模型规模增长的核心驱动力之一。



## L1和L2正则化分别是什么，什么场景适合使用呢？

L1和L2正则化都是在机器学习和深度学习中用于防止模型过拟合的常用技术。它们通过在模型的损失函数（Loss Function）中添加一个代表模型复杂度的惩罚项来实现这一目标。

#### L1 正则化 (L1 Regularization / Lasso)

* <strong>定义：</strong> L1正则化添加的惩罚项是模型所有权重参数 $w_i$ 的<strong>绝对值之和</strong>，乘以一个正则化系数 $\lambda$。
  $$
  \text{Loss}_{L1} = \text{Original Loss} + \lambda \sum_{i} |w_i|
  $$

* <strong>核心作用：产生稀疏性 (Sparsity)</strong>。
  在梯度下降优化过程中，L1惩罚项会驱使那些对模型贡献不大的特征的权重最终变为<strong>精确的0</strong>。这相当于从模型中完全移除了这些特征。

* <strong>适用场景：特征选择 (Feature Selection)</strong>。
  当你的数据集中包含大量特征，但你怀疑其中许多特征是冗余或无用的时，L1正则化非常有用。它能够自动地“筛选”出最重要的特征，简化模型，提高解释性。

#### L2 正则化 (L2 Regularization / Ridge / Weight Decay)

* <strong>定义：</strong> L2正则化添加的惩罚项是模型所有权重参数 $w_i$ 的<strong>平方和</strong>，乘以一个正则化系数 $\lambda$。
  $$
  \text{Loss}_{L2} = \text{Original Loss} + \lambda \sum_{i} w_i^2
  $$

* <strong>核心作用：权重衰减 (Weight Decay)</strong>。
  L2正则化会惩罚大的权重值，它会促使模型的权重参数尽可能小，<strong>趋近于0但通常不会等于0</strong>。这使得模型的权重分布更加平滑和分散，避免模型过度依赖少数几个高权重的特征。

* <strong>适用场景：通用性的过拟合防治</strong>。
  L2是更常用、更通用的正则化方法。当特征之间可能存在相关性（共线性），或者你认为绝大多数特征都对预测有或多或少的贡献时，L2是首选。它能有效地提高模型的泛化能力，使其在未见过的数据上表现更好。在深度学习中，“权重衰减”通常就是指L2正则化。

<strong>总结对比：</strong>

| 对比项                    | L1 正则化                              | L2 正则化                |
| :------------------------ | :------------------------------------- | :----------------------- |
| <strong>惩罚项</strong>   | 权重的绝对值之和 (L1范数)              | 权重的平方和 (L2范数)    |
| <strong>效果</strong>     | 权重稀疏化，部分权重为0                | 权重平滑化，权重趋近于0  |
| <strong>主要用途</strong> | 特征选择，简化模型                     | 防止过拟合，提升泛化能力 |
| <strong>解的特性</strong> | 不稳定，数据微小变动可能导致特征集变化 | 稳定，解是唯一的         |



## 激活函数有了解吗，你知道哪些LLM常用的激活函数？

激活函数是神经网络中至关重要的一环，它的主要作用是<strong>为网络引入非线性（non-linearity）</strong>。如果没有激活函数，多层神经网络本质上等同于一个单层的线性模型，无法学习和拟合复杂的数据模式。

在现代大型语言模型（Transformer架构）中，最常用的激活函数主要有两个：<strong>GeLU</strong> 和 <strong>SwiGLU</strong>。

1.  <strong>GeLU (Gaussian Error Linear Unit):</strong>
    * <strong>简介：</strong> GeLU曾是Transformer模型中的主流激活函数，被BERT、GPT-2等经典模型采用。它的数学形式是 $x \cdot \Phi(x)$，其中 $\Phi(x)$ 是高斯分布的累积分布函数。
    * <strong>为什么选用它？</strong>
      * <strong>平滑性：</strong> GeLU是ReLU的一个平滑近似。相比于ReLU在0点的突变，GeLU的平滑特性使其在优化过程中梯度更稳定，更有利于模型收敛。
      * <strong>随机正则化思想：</strong> GeLU可以看作是综合了Dropout和ReLU的思想。它根据输入的数值大小，对其进行随机的“归零”或“保留”，但这个过程是确定性的。输入越小，其输出被“归零”的概率越高。

2.  <strong>SwiGLU (Swish-Gated Linear Unit):</strong>
    * <strong>简介：</strong> SwiGLU是目前<strong>最先进、最主流</strong>的选择，被Llama、PaLM、Mixtral、Gemma等一系列现代LLM广泛采用。它属于<strong>门控线性单元（Gated Linear Unit, GLU）</strong> 家族的变体。
    * <strong>工作原理：</strong> 它将前馈网络（FFN）的第一个线性层的输出 $X$ 分成两部分， $A$ 和 $B$ 。然后通过公式 $Swish(A) \otimes B$ 计算输出，其中 $Swish(x) = x \cdot \sigma(x)$ ， $\sigma$ 是Sigmoid函数， $\otimes$ 是逐元素相乘。
    * <strong>为什么选用它？</strong>
      * <strong>门控机制（Gating Mechanism）：</strong> SwiGLU的核心优势在于其“门控”设计。 $B$ 部分可以被看作一个动态的“门”，它可以根据输入内容，控制 $Swish(A)$ 中的信息哪些可以通过、哪些需要被抑制。这种机制<strong>显著增强了模型的表达能力</strong>，使得FFN层可以更灵活地处理信息。
      * <strong>实证效果优越：</strong> Google在PaLM论文中的实验发现，使用SwiGLU替换标准的GeLU或ReLU，可以<strong>显著提升模型的性能</strong>（降低困惑度）。尽管SwiGLU会增加FFN层的参数量（因为需要两个矩阵而不是一个），但其带来的性能增益被证明是值得的。

## 混合专家模型（MoE）是如何在不显著增加推理成本的情况下，有效扩大模型参数规模的？请简述其工作原理。

混合专家模型（Mixture of Experts, MoE）是一种模型架构，它的核心思想是通过 <strong>“稀疏激活”（Sparse Activation）</strong> 的策略，来解决模型规模与计算成本之间的矛盾。它允许模型拥有巨大的总参数量，但在处理任何一个输入时，只动用其中一小部分参数，从而在不显著增加推理成本（FLOPs）的情况下，大幅提升模型容量。

<strong>工作原理如下：</strong>

1.  <strong>用“专家”替换FFN层：</strong>
    * 在标准的Transformer架构中，计算量最大的部分之一是前馈网络（Feed-Forward Network, FFN）层。
    * MoE架构将模型中的部分或全部FFN层替换为<strong>MoE层</strong>。一个MoE层由两部分组成：
      * <strong>N个“专家”（Experts）：</strong> 每个专家本身就是一个独立的、规模较小的FFN。
      * <strong>1个“门控网络”或“路由器”（Gating Network / Router）：</strong> 这是一个小型的神经网络，通常是一个简单的线性层。

2.  <strong>动态路由决策：</strong>
    * 当一个词元（token）的向量表示来到MoE层时，它首先被送入<strong>路由器</strong>。
    * 路由器的作用是 <strong>“决策”</strong> ，判断这个token应该由哪些专家来处理最合适。它会输出一个包含N个分数的向量，代表该token与N个专家的“匹配度”。

3.  <strong>Top-K稀疏激活：</strong>
    * 路由器输出的分数经过Softmax归一化后，系统并<strong>不会</strong>激活所有的专家。相反，它只选择分数<strong>最高的Top-K个专家</strong>（K通常很小，比如1或2）。
    * 这就是“稀疏激活”的关键：对于每一个token，只有极少数（K个）专家被激活并进行计算，其余的（N-K个）专家则完全不参与，不产生任何计算成本。

4.  <strong>加权输出：</strong>
    * 被选中的K个专家分别对输入的token向量进行处理，得到K个输出向量。
    * 最终的输出是这K个输出向量的<strong>加权和</strong>，权重同样由路由器的输出分数决定。

<strong>如何实现“参数大但成本低”？</strong>

* 假设一个模型有8个专家（N=8），并且每次只激活2个（K=2），如Mixtral-8x7B模型。
* <strong>总参数量：</strong> 模型的总参数量是所有共享部分（如注意力层）的参数量，加上<strong>所有8个专家</strong>的参数量之和。这使得模型的总参数规模可以非常大（例如达到47B）。
* <strong>推理成本：</strong> 但在进行一次前向传播（推理）时，对于任意一个token，实际参与计算的只有共享部分和<strong>被激活的2个专家</strong>。因此，其计算量（FLOPs）约等于一个规模小得多的“稠密”模型（例如一个13B的模型）。
* <strong>结论：</strong> MoE成功地将<strong>总参数量</strong>（代表模型的知识容量）和<strong>单次推理的计算量</strong>（代表模型的速度和成本）<strong>解耦</strong>，从而实现了“用小模型的成本，获得大模型的知识”。

## 在训练一个百或千亿参数级别的 LLM 时，你会面临哪些主要的工程和算法挑战？（例如：显存、通信、训练不稳定性等）

训练百亿或千亿参数级别的LLM是一个巨大的系统工程，涉及硬件、软件和算法的深度协同。其挑战主要体现在以下三个方面：

1. **显存挑战 (Memory Wall):**

* <strong>问题：</strong> 一个千亿参数的模型，其模型参数、梯度、优化器状态（如Adam中的动量和方差）加起来需要数TB的存储空间，远远超出了任何单张GPU的显存（目前最先进的H100也只有80GB）。
* <strong>解决方案（3D并行）：</strong>
  * <strong>数据并行 (Data Parallelism, DP):</strong> 最基础的并行方式。在每张卡上都保留一份完整的模型副本，但将数据切分成多个batch，每张卡处理一个batch。计算完成后通过All-Reduce操作同步梯度。这种方式<strong>不能解决单卡显存不足</strong>的问题。
  * <strong>流水线并行 (Pipeline Parallelism, PP):</strong> 将模型的层（layers）进行垂直切分，不同的GPU负责模型的一部分（例如，GPU-1负责1-16层，GPU-2负责17-32层）。这<strong>可以有效降低单卡显存</strong>，但会引入“流水线气泡”（pipeline bubbles），即部分GPU在等待上下游数据时会处于空闲状态。
  * <strong>张量并行 (Tensor Parallelism, TP):</strong> 将模型中的单个大算子（如大的权重矩阵）进行水平切分，放到不同的GPU上协同计算。例如，将一个大的矩阵乘法分解到多张卡上。这也能<strong>降低单卡显存</strong>，但会引入<strong>非常高</strong>的通信开销。
  * <strong>ZeRO (Zero Redundancy Optimizer):</strong> 由微软DeepSpeed提出的显存优化技术。它在数据并行的基础上，将<strong>优化器状态、梯度、甚至模型参数</strong>也进行切分，分布到所有GPU上。每个GPU只保留自己需要计算的那一部分，极大地降低了单卡的显存冗余，是目前大规模训练的标配。

2. **通信挑战 (Communication Bottleneck):**

* <strong>问题：</strong> 上述所有并行策略都引入了大量的GPU间通信。例如，DP需要同步梯度，PP需要传递激活值，TP需要在每次前向和后向传播中交换计算结果。当GPU数量巨大时，通信所需的时间可能超过计算本身，成为整个训练的瓶颈。
* <strong>解决方案：</strong>
  * <strong>硬件层面：</strong> 使用高速互联技术，如单机内的<strong>NVLink</strong>和跨节点的<strong>InfiniBand</strong>网络。
  * <strong>软件层面：</strong> 开发高效的通信算法（如Ring All-Reduce），并设计调度策略来将<strong>计算和通信操作重叠（overlap）</strong>，以隐藏通信延迟。

<strong>3. 训练不稳定性挑战 (Training Instability):</strong>

* <strong>问题：</strong> 训练如此巨大的模型在数值上非常脆弱。由于计算层数极深、数据量极大，训练过程中很容易出现<strong>梯度爆炸或消失</strong>，导致损失（Loss）突然飙升为NaN（Not a Number），使得数小时甚至数天的训练成果毁于一旦。
* <strong>解决方案：</strong>
  * <strong>数值精度：</strong> 普遍采用 <strong>BF16 (BFloat16)</strong> 混合精度训练。BF16相比FP16有更大的动态范围，能有效避免梯度下溢，同时保持FP32的稳定性。同时，关键部分（如优化器的master weights）仍保留FP32以保证精度。
  * <strong>稳定的模型架构：</strong> 采用更稳定的架构设计，如<strong>Pre-LayerNorm</strong>（在自注意力和FFN之前进行层归一化），以及使用更平滑的激活函数如<strong>GeLU/SwiGLU</strong>。
  * <strong>梯度裁剪 (Gradient Clipping):</strong> 设定一个梯度的范数上限，如果计算出的梯度超过这个阈值，就将其缩放到阈值以内，这是防止梯度爆炸最直接有效的方法。
  * <strong>学习率调度与预热 (Learning Rate Scheduling & Warmup):</strong> 采用精心设计的学习率调度策略，如在训练初期使用一个较小的学习率并逐渐增大的“预热”阶段，有助于模型在训练早期稳定下来。











