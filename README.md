# Flash3D: 单张图像的前馈式通用3D场景重建

## Stanislaw Szymanowicz\*
VGG, 牛津大学  
stan@robots.ox.ac.uk

## Eldar Insafutdinov\*
VGG, 牛津大学  
eldar@robots.ox.ac.uk

## Chuanxia Zheng\*
VGG, 牛津大学  
cxzheng@robots.ox.ac.uk

## Dylan Campbell
澳大利亚国立大学  
dylan.campbell@anu.edu.au

## João F. Henriques
VGG, 牛津大学  
joao@robots.ox.ac.uk

## Christian Rupprecht
VGG, 牛津大学  
chrisr@robots.ox.ac.uk

## Andrea Vedaldi
VGG, 牛津大学  
vedaldi@robots.ox.ac.uk

## 摘要
在本文中，我们提出了Flash3D，这是一种场景重建和新视图合成的方法，只需要单张图像，且具备很强的通用性和高效性。为了通用性，我们从单目深度估计的“基础”模型开始，并将其扩展为完整的3D形状和外观重建器。为了高效性，我们基于前馈高斯散射进行扩展。具体来说，我们首先在预测深度处预测3D高斯分布，然后添加偏移的高斯分布层，允许模型在遮挡和截断之间完成重建。Flash3D非常高效，在单个GPU上一天内即可训练完毕，因此大多数研究人员都能使用它。它在训练并在RealEstate10k上测试时取得了最先进的结果。当转移到未见过的数据集NYU时，它以很大优势超越了竞争对手。更为显著的是，当转移到KITTI时，Flash3D在PSNR上超过了专门针对该数据集训练的方法。在某些情况下，它甚至超越了使用多视图输入的最新方法。代码、模型、演示和更多结果可在[此处](https://www.robots.ox.ac.uk/~vgg/research/flash3d/)获得。

## 1 介绍
我们考虑通过网络的一次前馈传递从单张图像重建真实感3D场景的问题。这是一个具有挑战性的任务，因为场景复杂且单目重建是不适定的。单目设置中缺乏明确的几何线索，如三角测量，并且无法直接获得被遮挡部分的视图。

该问题与单目深度估计密切相关\[4, 6, 7, 16, 20, 33, 34, 46, 47, 56, 59, 90\]，这是一项成熟的技术。现在可以通过出色的跨域泛化\[47, 90, 91\]准确估计度量深度。然而，尽管深度估计器可以预测最近可见表面的3D形状，但它们并不提供任何外观信息，或者被遮挡或视野外部分的估计。仅深度不足以解决诸如新视图合成（NVS）等任务，这些任务还需要解决局部和视图依赖的外观。

尽管现代单目场景重建方法存在\[36, 74, 85\]，但它们大多在封闭环境中操作，即为每个特定数据集训练单独的网络。与之相反，现代的深度预测器在推理时对新数据集的泛化能力很好。此外，当前的单目场景重建器通常较慢或由于体积渲染\[36\]和隐式表示\[92\]而导致计算内存成本较高。

最近，Szymanowicz等人\[68\]介绍了Splatter Image（SI），这是一种基于高斯散射\[31\]成功的快速单目重建方法。该方法的实现很简单：为每个输入图像像素预测彩色3D高斯参数，使用标准的图像到图像神经网络架构。得到的高斯混合模型可以很好地重建对象，包括未观察到的表面。部分原因是SI可以使用一些“背景像素”来建模被遮挡的部分。然而，在场景重建中，并没有这样的背景像素储备，这对方法构成了挑战。相比之下，pixelSplat\[9\]，MVSplat\[11\]，latentSplat\[83\]和GS-LRM\[95\]，这些方法虽然共享类似的设计，旨在进行场景重建；然而，它们解决的是双目重建问题，需要两张已知视点的场景图像。我们考虑的是更具挑战性的单目设置，因为它更具普遍适用性且不需要相机外参，这本身就是一个具有挑战性的研究问题\[78, 80, 94\]。

在这项工作中，我们介绍了一种新的、简单、高效且高性能的单目场景重建方法，称为Flash3D。这基于两个关键理念。首先，我们解决了当前前馈单目场景重建器存在的通用性问题。目标是让Flash3D在任何场景中工作，而不仅仅是与训练集相似的场景。类似地，开放式模型通常称为基础模型，需要大量训练数据集和大多数研究小组无法使用的计算资源。3D重建和生成中也存在类似的问题\[35, 37, 40, 41, 60, 97\]，通过扩展到3D或使用现有基础2D图像或视频生成器\[5, 13, 19, 48, 53\]来解决。我们在此假设重建还可以从现有基础模型构建，但选择单目深度预测器作为更自然的选择。特别是，通过构建一个高质量的深度预测器\[47\]，我们可以实现对新数据集的出色泛化，以至于我们的3D重建比那些专门针对测试数据集训练的模型更准确。

其次，我们改进了用于单目场景重建的前馈逐像素高斯散射。正如所述，应用于单个对象，逐像素重建器可以使用背景像素建模对象的隐藏部分，但在重建整个场景时不可行。我们的解决方案是预测每个像素的多个高斯，其中只有第一个高斯沿光线一致以符合深度估计，从而建模场景的可见部分。这类似于分层表示\[1, 3, 36, 58, 74, 76\]和多高斯采样在pixelSplat\[9\]中的应用。然而，在我们的案例中，高斯是确定性的，不限于特定深度范围，模型是自由移动高斯以建模遮挡或截断的场景部分。

总体而言，Flash3D是一个简单且高性能的单目场景重建管道。经验表明，我们发现Flash3D可以(a)渲染高质量的重建3D场景图像；(b)在广泛的场景中操作，包括室内和室外；并且(c)重建被遮挡部分，这在仅深度估计或简单的扩展中是不可能实现的。Flash3D在所有指标上在RealEstate10K\[72\]上实现了最先进的新视图合成精度。
更为显著的是，相同的冻结模型在转移到NYU\[61\]和KITTI\[18\]时也能达到最先进的精度（在PSNR上）。此外，在外推设置中，我们的重建甚至比使用两张场景图像而非一张的双目方法（如pixelSplat\[9\]和latentSplat\[83\]）更准确，因此具有显著的优势。

<p align="center">
  <img width="639" alt="image" src="https://github.com/user-attachments/assets/de7a0749-07ec-4312-9f40-2cbdeff5cb92">
</p>

除了重建的质量，如图1所示，Flash3D在评估和，尤其是训练方面非常高效。例如，我们使用的是MINE\[36\]的GPU资源的1/64。通过使用较少的计算资源实现最先进的结果，这为更广泛的研究人员打开了研究领域的大门。

## 2 相关工作

### 单目前馈重建
与我们的方法类似，单目前馈重建器通过神经网络传递单张场景图像以直接输出3D重建。对于场景，\[74-76\]和MINE\[36\]的方法通过预测多平面图像\[72\]，\[85\]则使用神经辐射场\[43, 62\]。我们的方法在速度和泛化能力上优于它们。与我们的工作类似，SynSin\[84\]使用单目深度预测器重建场景；然而，它的重建是不完整的，需要渲染网络来改进最终视图。相比之下，我们的方法输出高质量的3D重建，可以直接用高斯散射\[31\]渲染。对于对象，一个显著的例子是大重建模型（LRM）\[27\]，它获得高质量的单目重建，但模型非常大，训练成本非常高。与我们最相关的工作是使用高斯散射\[31\]的Splatter Image\[68\]以提高效率。我们的方法也使用高斯散射作为表示，但这样做是为了场景而非对象，这提出了不同的挑战。

### 少视图前馈重建
一个较少但仍然重要的案例是少视图前馈重建，其中重建需要从已知视点的两张或更多图像中进行。早期示例使用神经辐射场（NeRF）\[43\]作为对象\[12, 25, 29, 38, 51, 79, 92\]和场景\[12, 14, 88\]的3D表示。这些方法隐式地学习视点之间的匹配点；\[10, 93\]的方法使点匹配更加明确。虽然许多少视图重建器通过显式字段估计对象的3D形状，但另一种选择是直接预测新视图\[44, 55, 65, 66\]或没有显式体积重建的场景\[63\]，这是光场网络\[63\]的先驱概念。其他工作使用多平面图像从窄基线立体对\[64, 72\]和少视图\[30, 42\]重建。与我们的方法类似，pixelSplat\[9\]，latentSplat\[83\]和MVSplat\[11\]从一对图像重建场景。它们使用跨视图注意力有效地共享信息，并预测高斯混合以表示场景几何。最近的前馈方法（例如\[69, 89, 95\]）结合LRM和高斯散射从少量图像中进行重建。我们解决单目重建问题，这更具挑战性，因为缺乏三角测量的几何线索。

### 迭代重建
迭代或优化方法从一张或多张图像重建，通过反复拟合3D模型来达到数据。这类方法由于其迭代性质，以及需要倾斜3D模型以适应数据，通常比前馈方法慢得多。DietNeRF\[28\]使用语言模型进行正则化重建，RegNeRF\[45\]和RefNeRF\[77\]使用手工正则化，SinNeRF\[87\]使用单目深度。RealFusion\[40\]使用图像扩散模型作为单目重建的先验，基于慢速课程蒸馏采样迭代\[49\]。许多后续工作遵循类似的路径\[70, 86\]。融合生成和鲁棒性可以通过使用多视图感知生成器改进\[23, 37, 41, 73, 82, 97\]。Viewset Diffusion\[67\]和RenderDiffusion\[2\]方法融合3D重建和扩散生成，可以减少但不能消除迭代生成的成本。相比之下，我们的方法是前馈的，因此显著更快，接近实时（10fps）。一些方法以前馈方式生成新视图，但以迭代和自回归方式生成一个视图。示例包括PixelSynth\[52\]，扩展SynSin，GeNVS\[8\]，和Text2Room\[26\]。相比之下，我们在一次前馈传递中生成最终的3D重建。

### 单目深度预测
我们的方法基于单目深度估计\[4, 6, 7, 15, 16, 20, 21, 33, 34, 46, 50, 56, 59, 90, 98\]，这些方法预测每个像素的度量或相对深度。通过从大数据集的图像中学习深度，有些方法展示了高精度和跨域泛化的能力。虽然我们的方法与深度预测器紧密结合，但我们使用一种最先进的度量深度估计器UniDepth\[47\]进行实验。

更为显著的是，相同的冻结模型在转移到NYU\[61\]和KITTI\[18\]时也能达到最先进的精度（在PSNR上）。此外，在外推设置中，我们的重建甚至比使用两张场景图像而非一张的双目方法（如pixelSplat\[9\]和latentSplat\[83\]）更准确，因此具有显著的优势。

除了重建的质量，如图1所示，Flash3D在评估和，最重要的是，训练方面非常高效。例如，我们使用的是MINE\[36\]的GPU资源的1/64。通过使用较少的计算资源实现最先进的结果，这为更广泛的研究人员打开了研究领域的大门。

## 2 相关工作

### 单目前馈重建
与我们的方法类似，单目前馈重建器通过神经网络传递单张场景图像以直接输出3D重建。对于场景，\[74-76\]和MINE\[36\]的方法通过预测多平面图像\[72\]，\[85\]则使用神经辐射场\[43, 62\]。我们的方法在速度和泛化能力上优于它们。与我们的工作类似，SynSin\[84\]使用单目深度预测器重建场景；然而，它的重建是不完整的，需要渲染网络来改进最终视图。相比之下，我们的方法输出高质量的3D重建，可以直接用高斯散射\[31\]渲染。对于对象，一个显著的例子是大重建模型（LRM）\[27\]，它获得高质量的单目重建，但模型非常大，训练成本非常高。与我们最相关的工作是使用高斯散射\[31\]的Splatter Image\[68\]以提高效率。我们的方法也使用高斯散射作为表示，但这样做是为了场景而非对象，这提出了不同的挑战。

### 少视图前馈重建
一个较少但仍然重要的案例是少视图前馈重建，其中重建需要从已知视点的两张或更多图像中进行。早期示例使用神经辐射场（NeRF）\[43\]作为对象\[12, 25, 29, 38, 51, 79, 92\]和场景\[12, 14, 88\]的3D表示。这些方法隐式地学习视点之间的匹配点；\[10, 93\]的方法使点匹配更加明确。虽然许多少视图重建器通过显式字段估计对象的3D形状，但另一种选择是直接预测新视图\[44, 55, 65, 66\]或没有显式体积重建的场景\[63\]，这是光场网络\[63\]的先驱概念。其他工作使用多平面图像从窄基线立体对\[64, 72\]和少视图\[30, 42\]重建。与我们的方法类似，pixelSplat\[9\]，latentSplat\[83\]和MVSplat\[11\]从一对图像重建场景。它们使用跨视图注意力有效地共享信息，并预测高斯混合以表示场景几何。最近的前馈方法（例如\[69, 89, 95\]）结合LRM和高斯散射从少量图像中进行重建。我们解决单目重建问题，这更具挑战性，因为缺乏三角测量的几何线索。

### 迭代重建
迭代或优化方法从一张或多张图像重建，通过反复拟合3D模型来达到数据。这类方法由于其迭代性质，以及需要倾斜3D模型以适应数据，通常比前馈方法慢得多。DietNeRF\[28\]使用语言模型进行正则化重建，RegNeRF\[45\]和RefNeRF\[77\]使用手工正则化，SinNeRF\[87\]使用单目深度。RealFusion\[40\]使用图像扩散模型作为单目重建的先验，基于慢速课程蒸馏采样迭代\[49\]。许多后续工作遵循类似的路径\[70, 86\]。融合生成和鲁棒性可以通过使用多视图感知生成器改进\[23, 37, 41, 73, 82, 97\]。Viewset Diffusion\[67\]和RenderDiffusion\[2\]方法融合3D重建和扩散生成，可以减少但不能消除迭代生成的成本。相比之下，我们的方法是前馈的，因此显著更快，接近实时（10fps）。一些方法以前馈方式生成新视图，但以迭代和自回归方式生成一个视图。示例包括PixelSynth\[52\]，扩展SynSin，GeNVS\[8\]，和Text2Room\[26\]。相比之下，我们在一次前馈传递中生成最终的3D重建。

### 单目深度预测
我们的方法基于单目深度估计\[4, 6, 7, 15, 16, 20, 21, 33, 34, 46, 50, 56, 59, 90, 98\]，这些方法预测每个像素的度量或相对深度。通过从大数据集的图像中学习深度，有些方法展示了高精度和跨域泛化的能力。虽然我们的方法与深度预测器紧密结合，但我们使用一种最先进的度量深度估计器UniDepth\[47\]进行实验。

<p align="center">
<img width="627" alt="image" src="https://github.com/user-attachments/assets/b3b5126e-3815-4f2b-8704-e6e2df125619">
</p>

如图2所示，Flash3D的概述。给定单张图像 \( I \) 作为输入，Flash3D首先使用冻结的现成网络\[47\]估计度量深度 \( D \)。然后，类似ResNet50的编码-解码网络预测每个像素 \( u \) 的 \( K \) 层高斯的形状和外观参数集合 \( \mathcal{P} \)，允许建模未观察到和被遮挡的表面。从这些预测的成分中，可以通过将预测的（正）偏移 \( \delta_i \) 与预测的单目深度 \( D \) 相加来获得每层高斯的均值向量。这种策略确保各层深度有序，鼓励网络建模被遮挡的表面。

## 3 方法

设

$$
I \in \mathbb{R}^{3 \times H \times W}
$$

是一个RGB图像。我们的目标是学习一个神经网络 

$$
\Phi
$$

它将

$$
I
$$

作为输入并预测场景的3D内容表示 

$$
G = \Phi(I)
$$

包括3D几何和光度。我们首先在第3.1节讨论背景和基线模型，然后在第3.2节介绍我们分层多高斯预测器，并在第3.2节讨论使用单目深度预测作为先验。

### 3.1 背景：从单张图像重建场景

#### 表示：作为3D高斯集合的场景

场景表示 

$$
G = \{(\sigma_i, \mu_i, \Sigma_i, c_i)\}_{i=1}^G
$$

是一组3D高斯\[31\]，其中 

$$
\sigma_i \in [0, 1]
$$

是不透明度，

$$
\mu_i \in \mathbb{R}^3
$$

是均值，

$$
\Sigma_i \in \mathbb{R}^{3 \times 3}
$$

是协方差矩阵，

$$
c_i : S^2 \rightarrow \mathbb{R}^3
$$

是每个成分的方向颜色。设

$$
g_i(x) = \exp \left( -\frac{1}{2} (x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i) \right)
$$

为对应的（非归一化）高斯函数。高斯的颜色通常用球谐函数表示，所以

$$
[c_i(\nu)]_j = \sum_{l=0}^{L-1} \sum_{m=-l}^l c_{ijlm} Y_{lm}(\nu)
$$

其中 

$$
\nu \in S^2
$$

是一个方向，

$$
Y_{lm}
$$

是各种顺序和次数的球谐函数。高斯混合

$$
\sigma(x) = \frac{\sum_{i=1}^G \sigma_i g_i(x)}{\sum_{i=1}^G \sigma_i g_i(x)}
$$

线性组合颜色和辐射场的颜色函数：

$$
c(x, \nu) = \frac{\sum_{i=1}^G \sigma_i g_i(x) c_i(\nu)}{\sum_{i=1}^G \sigma_i g_i(x)}
$$

其中 

$$
\sigma(x)
$$

是 

$$
\mathbb{R}^3
$$

中位置 

$$
x
$$

的3D不透明度，

$$
c(x, \nu)
$$

是朝向相机方向

$$
\nu
$$

的

$$
x
$$

处的辐射。

通过使用辐射-吸收方程\[39\]积分辐射来渲染图像

$$
J(u)
$$

即

$$
J(u) = \int_0^\infty \sigma(x_t, \nu) c(x_t, \nu) \exp \left( - \int_0^t \sigma(x_\tau) d\tau \right) dt
$$

其中 

$$
x_t = x_0 - t \nu
$$

是从相机中心 

$$
x_0
$$

出发沿像素 

$$
u
$$

的方向传播的光线。高斯散射\[31\]的关键贡献是非常有效地近似此积分，实施可微分渲染函数 

$$
\hat{J} = \text{Rend}(G, \pi)
$$

将高斯混合 

$$
G
$$

和视点 

$$
\pi
$$

作为输入并返回相应图像的估计 

$$
\hat{J} 。
$$



#### 单目重建

根据\[68\]，神经网络的输出 

$$
\Phi(I) \in \mathbb{R}^{C \times H \times W}
$$

是一个张量，指定每个像素 

$$
u = (u_x, u_y, 1)
$$

的彩色高斯参数，包括不透明度 

$$
\sigma ，
$$

深度 

$$
d \in \mathbb{R}_+ ，
$$

偏移 

$$
\Delta \in \mathbb{R}^3 ，
$$

协方差 

$$
\Sigma \in \mathbb{R}^{3 \times 3}
$$

表示旋转和缩放（使用四元数表示旋转），以及颜色模型 

$$
c \in \mathbb{R}^{3(L+1)^2} ，
$$

其中 

$$
L
$$

是球谐函数的顺序。高斯的均值为

$$
\mu = K^{-1} u d + \Delta
$$

其中 

$$
K = \text{diag}(f, f, 1) \in \mathbb{R}^{3 \times 3}
$$

是相机校准矩阵，

$$
f
$$

是焦距。因此，每个像素预测

$$
C = 1 + 1 + 3 + 7 + 3(L+1)^2 = 12 + 3(L+1)^2
$$

参数。模型 

$$
\Phi
$$

使用三元组 

$$
(J, I, \pi)
$$

进行训练，其中 

$$
I
$$

是目标图像，

$$
J
$$

是目标图像，

$$
\pi
$$

是相对相机姿态。为了学习网络参数，只需最小化渲染损失

$$
L(G, \pi, J) = ||\text{Rend}(G, \pi) - J||
$$

### 3.2 单目前馈多高斯

为了通用性，我们建议在高质量的预训练模型上构建Flash3D，该模型在大量数据上训练。特别是，鉴于单目场景重建和单目深度估计之间的相似性，我们使用一个现成的单目深度预测器 

$$
\Psi 。
$$


该模型将图像

$$
I
$$

作为输入并返回深度图

$$
D = \Psi(I)
$$

其中

$$
D \in \mathbb{R}_+^{H \times W}
$$

是一个深度值矩阵，如下所述。

### 基线架构

给定图像

$$
I
$$

和估计的深度图

$$
D
$$

我们的基线模型由一个附加的网络

$$
\Phi(I, D)
$$

组成，该网络将图像和深度图作为输入并返回每像素高斯参数的集合。具体来说，对于每个像素

$$
u
$$

实体

$$
\left[ \Phi(I, D) \right]_u = (\sigma, \Delta, s, \theta, c)
$$

包括不透明度

$$
\sigma \in \mathbb{R}_+
$$

位移

$$
\Delta \in \mathbb{R}^3
$$

尺度

$$
s \in \mathbb{R}
$$

四元数

$$
\theta \in \mathbb{R}^4
$$

用于旋转参数化

$$
R(\theta)
$$

以及颜色参数

$$
c
$$

每个高斯的协方差由

$$
\Sigma = R(\theta)^\top \text{diag}(s) R(\theta)
$$

给出，均值由

$$
\mu = (u_x d / f, u_y d / f, d) + \Delta
$$

给出，其中

$$
f
$$

是相机的焦距（已知或由

$$
\Psi
$$

估计），深度

$$
d = D(u)
$$

来自深度图。网络

$$
\Phi
$$

是一个使用ResNet块的U-Net\[54\]，用于编码和解码，分别记作

$$
\Phi_{\text{enc}}
$$

和

$$
\Phi_{\text{dec}}
$$

解码器网络输出张量

$$
\Phi_{\text{dec}}(\Phi_{\text{enc}}(I, D)) \in \mathbb{R}^{(C-1) \times H \times W}
$$

请参阅补充材料了解详细信息。

### 多高斯预测

虽然模型中的高斯可能会从对应像素的光线偏移，但每个高斯倾向于建模投影到该像素的对象部分。Szymanowicz 等人\[68\]指出，对于单个对象，有大量不与任何对象表面相关的背景像素，这些可以被重新用于建模3D对象的未观察部分。然而，对于每个像素预测多个高斯来说，这是困难的，因为模型必须处理遮挡和超出图像视野的部分。因此，我们建议为每个像素预测一个小数量

$$
K > 1
$$

的不同高斯。

从概念上讲，给定图像

$$
I
$$

和估计的深度图

$$
D
$$

我们的网络为每个像素

$$
u
$$

预测一组形状、位置和外观参数

$$
\mathcal{P} = \{(\sigma_i, \delta_i, \Delta_i, \Sigma_i, c_i)\}_{i=1}^K
$$

其中第

$$
i
$$

个高斯的深度由下式给出：

$$
d_i = d + \sum_{j=1}^i \delta_j
$$

其中

$$
d = D(u)
$$

是深度图

$$
D
$$

中像素

$$
u
$$

处的预测深度，且

$$
\delta_1 = 0
$$

是常数。注意，由于深度偏移

$$
\delta_i
$$

不能为负，这确保后续的高斯层在前面的“后面”，并鼓励网络建模遮挡的表面。第

$$
i
$$

个高斯的均值由下式给出：

$$
\mu_i = (u_x d_i / f, u_y d_i / f, d_i) + \Delta_i
$$

实际上，我们发现

$$
K = 2
$$

是一个足够的表达。

### 超出边界的重建

如我们经验所示，对于网络能够建模刚刚超出视野的3D内容非常重要。虽然多个高斯层有助于此，但特别需要在图像边缘附近的附加高斯（例如，当相机缩回时的新视角合成）。为了便于获取这些高斯，编码器

$$
\Phi_{\text{enc}}
$$

通过在每侧填充

$$
P > 0
$$

像素的方式开始输入图像和深度

$$
(I, D)
$$

以使得输出

$$
\Phi_k(I, D) \in \mathbb{R}^{(C-1) \times (H+2P) \times (W+2P)}
$$

大于输入。

## 4 实验

我们设计实验以支持四个关键发现，每个部分专注于一个发现。我们从最重要的结果开始：跨数据集的泛化——利用单目深度预测网络并在单一数据集上训练可以在其他数据集上获得良好的重建质量（第4.2节）。其次，我们确定Flash3D作为单视图3D重建的有效表示，与专门为此任务设计的方法进行比较（第4.3节）。第三，我们甚至展示了单视图Flash3D学到的先验与双视图方法学到的一样强（第4.4节）。最后，我们通过消融研究展示了每个设计选择如何贡献于Flash3D的性能（第4.5节），并分析了Flash3D的输出以深入了解其内部工作原理。

### 4.1 实验设置

#### 数据集

表1：跨域新视图合成。我们评估在未用于训练我们方法的数据集上的新视图合成（NVS）准确性。我们在KITTI特定基线上优于其他方法。这里，跨域（CD）表示该方法未在所评估的数据集上进行训练。

| 方法             | CD  | PSNR↑ | SSIM↑ | LPIPS↓ | CD  | PSNR↑ | SSIM↑ | LPIPS↓ |
|-----------------|-----|-------|-------|--------|-----|-------|-------|--------|
| LDI [76]        | ✗   | 16.50 | 0.572 | -      | -   | -     | -     | -      |
| SV-MPI [74]     | ✗   | 19.50 | 0.733 | -      | -   | -     | -     | -      |
| BTS [85]        | ✗   | 20.10 | 0.761 | 0.144  | -   | -     | -     | -      |
| MINE [36]       | ✗   | 21.90 | 0.828 | 0.112  | ✓   | 24.33 | 0.745 | 0.202  |
| UniDepth w/U    | ✓   | 20.86 | 0.774 | 0.154  | ✓   | 22.54 | 0.732 | 0.212  |
| Flash3D (Ours)  | ✓   | 21.96 | 0.826 | 0.132  | ✓   | 25.45 | 0.774 | 0.196  |

训练和7,289次测试后，一旦Flash3D经过训练，我们评估其在各种数据集上的有效性，在每个部分中详细讨论。有关评估协议的详细信息，请参阅附录。

Flash3D仅在大规模的RealEstate10k\[72\]数据集上训练，该数据集包含来自YouTube的真实房地产视频。我们遵循默认的训练/测试划分，使用67,477个场景进行训练，7,289个场景进行测试。一旦Flash3D经过训练，我们会评估其在各种数据集上的有效性，详细内容将在各部分中讨论。有关评估协议的详细信息，请参阅附录。

### 度量标准

对于定量结果，我们报告标准的图像质量度量指标，包括像素级PSNR、块级SSIM和特征级LPIPS。

### 比较方法

我们与几种专门为单视图场景重建设计的最新方法进行了比较，包括LDI [76]、Single-View MPI [74]、SynSin [84]、BTS [85] 和MINE [36]。我们还包括一个对Splatter Image [68] 的改编比较，以表明我们的方法更适合于一般场景，并与深度网络预测的位置上的输入颜色的射线解投影 \( U \) 进行了比较。最后，虽然这不是一个公平的比较，但我们也与最新的双视图新视图合成方法进行了比较，包括[14]、pixelSplat [9]、MVSplat [11] 和 latentSplat [83]。

### 实现细节

Flash3D由一个预训练的UniDepth [47] 模型、一个ResNet50 [24] 编码器、多深度偏移解码器和高斯解码器组成。整个模型在一块A6000 GPU上训练40,000次迭代，批量大小为16。训练效率极高，在一块A6000 GPU上只需一天即可完成。鉴于UniDepth在训练过程中保持冻结状态，我们可以通过预先提取整个数据集的深度图来加速训练。通过这种方式，Flash3D可以在16小时内在一块A6000 GPU上训练到最先进的质量水平。

### 4.2 跨域新视图合成

数据集。为了评估跨域泛化能力，我们直接在未见过的户外（KITTI [17]）和室内（NYU [61]）数据集上评估性能。对于KITTI，我们遵循标准基准，使用1,079张图像进行测试。对于NYU，我们制定了一个新的协议，使用250个源图像进行测试（详见补充材料）。我们以相同方式评估所有方法。我们验证了UniDepth [47] 未在这些数据集上进行训练。

据我们所知，我们是第一个报告单目重建跨域前馈性能的。我们考虑了两个具有挑战性的比较。首先，我们在NYU数据集上评估Flash3D和当前最新方法[36]，一个类似于RE10k的室内数据集，但在训练中未见过。在表1中，我们观察到我们的方法在此传输实验中表现明显更好，尽管域间差异相对较小。这表明先前的工作在泛化方面不如我们的方法。其次，我们在表1中比较我们在KITTI上的方法，发现它的表现与在该数据集上训练的最新方法相当。事实上，Flash3D在PSNR方面优于其他方法，尽管仅在室内数据集上训练。这表明利用预训练的深度网络使我们的网络能够学习一个极其强大的形状和外观先验，比直接在该数据集上学习更准确。

### 4.3 域内新视图合成

表2：域内新视图合成。我们的模型在RealEstate10k的小、中、大基线范围内显示了最先进的域内性能。

| 模型              | 5 帧                  | 10 帧                | \( U[-30, 30] \) 帧      |
|-------------------|----------------------|---------------------|-------------------------|
|                   | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
| Syn-Sin [84]      | -       | -       | -        | -       | -       | -        | 22.30  | 0.740  | -        |
| SV-MPI [74]       | 27.10   | 0.870   | -        | 24.40   | 0.812   | -        | 23.52  | 0.785  | -        |
| BTS [85]          | -       | -       | -        | -       | -       | -        | 24.00  | 0.755  | 0.194    |
| Splatter Image [68]| 28.15   | 0.894   | 0.110    | 25.34   | 0.842   | 0.144    | 24.15  | 0.810  | 0.177    |
| MINE [36]         | 28.45   | 0.897   | 0.111    | 25.89   | 0.850   | 0.150    | 24.75  | 0.820  | 0.179    |
| Flash3D (Ours)    | 28.46   | 0.899   | 0.100    | 25.94   | 0.857   | 0.133    | 24.93  | 0.833  | 0.160    |


我们在RealEstate10k [72] 上进行域内评估，遵循之前工作的相同协议[36]。我们评估零镜头重建的质量，并比较域内数据集RealEstate10k上的性能。我们使用新视图合成指标评估重建质量，因为这是该数据集中唯一可用的地面真实数据。RealEstate10k评估在不同距离之间的重建质量，因为较小的距离使任务更容易。在表2中，我们观察到在所有距离上都实现了最先进的结果。进一步的分析（如图3所示）表明，尽管我们的方法仅在少量GPU（1 vs. 64）上训练，但其重建效果更清晰、更准确，优于先前的最新方法[36]。

<p align="center">
<img width="637" alt="image" src="https://github.com/user-attachments/assets/f4cd47da-dffc-42c6-8be4-5560da8639b8">
</p>

### 4.4 与少视图新视图合成的比较

数据集。为了进一步评估Flash3D的有效性，我们使用pixelSplat [9] 的拆分进行插值评估，并使用latentSplat [83] 的拆分进行外推评估。

与现有的双视图方法通常评估两个源视图之间的插值不同，Flash3D始终从单个视图进行外推。结果在表3中报告。这里，Flash3D在插值任务中无法超越双视图方法，因为接收到的信息更少。然而，在视图外推方面，Flash3D超越了所有之前最先进的双视图方法。这突显了我们方法中多层高斯表示在捕捉和建模未见区域方面的实用性。

表3：与双视图方法的比较。我们在由pixelSplat [9] 用于双视图插值的拆分和由latentSplat [83] 用于外推的拆分上进行比较。我们选择最接近目标的视图作为源视图。我们的方法使用单个视图仍能更好地进行外推。

| 方法                | 输入视图 | RE10k 插值                      | RE10k 外推                      |
|--------------------|---------|-------------------------------|-------------------------------|
|                    |         | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
| Du et al [14]      | 2       | 24.78  | 0.820  | 0.213    | 21.83  | 0.790  | 0.242    |
| pixelSplat [9]     | 2       | 26.09  | 0.864  | 0.136    | 21.84  | 0.777  | 0.216    |
| latentSplat [83]   | 2       | 23.93  | 0.812  | 0.164    | 22.62  | 0.777  | 0.196    |
| MVSplat [11]       | 2       | 26.39  | 0.869  | 0.128    | 23.04  | 0.813  | 0.185    |
| Flash3D (Ours)     | 1       | 23.87  | 0.811  | 0.185    | 24.10  | 0.815  | 0.185    |

### 4.5 消融研究和分析

#### 消融研究

我们在表4中对我们的方法进行了域内和跨域设置的消融研究，重点关注以下问题。问题1：在场景外观和几何重建任务中，利用单目深度预测器是否有用？问题2：如果是，它本身是否足够，即是否有必要学习形状和外观参数以用3D高斯重建场景？

#### 深度预测器的重要性

我们移除了预测深度D的预训练深度网络，而是将其与所有其他参数一起联合估计。首先，在表4中我们观察到，与Flash3D相比，这导致性能显著下降，这表明深度网络包含了已学习的重要线索。此外，表4的第三行表明，在没有深度网络的情况下，每像素使用2层高斯的性能比仅使用一层更差。我们假设深度网络在避免限制基于原始方法的学习能力的局部最优方面起到了重要作用[9]。定性地讲，图4的第四列说明了移除深度网络使得学习准确的墙壁几何（橙色墙壁弯曲）和对象边界（床形状不正确）变得具有挑战性。

#### 超越深度预测的扩展重要性

这里我们移除了我们网络的学习部分。首先，我们仅使用一层高斯，并预测与预训练深度网络预测的深度D对应的参数P1。这也导致了表4中的性能下降。然后，我们进一步移除了预测P1的网络，完全取消了学习。新视图仅从源视图颜色通过深度D反向投影进行渲染。这显然进一步降低了性能。在图4中，我们观察到这些下降是由于一层方法无法表示场景的遮挡部分。图4的最后一列显示，当仅使用深度反投影时，孔洞是显著的，而通过学习形状P1可以部分缓解这一问题，因为网络能够在深度不连续处拉伸高斯。

#### 分析

图5分析了高斯层形成完整场景重建的机制。我们可视化了每层的深度D和D + delta_2，乘以相应高斯的不透明度igma_1, sigma_2，说明了它们在渲染场景时的使用程度。在图5中，颜色越饱和，高斯越不透明（黑色是完全透明）。我们观察到，第一层在对象边界（墙壁、柜子）和复杂几何形状（椅子）处具有最不透明的高斯，这表明这些是深度预测网络最有用的区域。这在图4中得到了进一步支持，其中移除深度网络在完全相同的区域内损害了重建。在对象边界利用深度网络导致小基线下的清晰、准确几何形状（第四列）。有趣的是，网络忽略了窗口的深度预测，这些预测始终不正确。接下来，我们分析第二层高斯放置其预测的位置。在图5的第三列中，当网络观察到一堵墙时，第二层高斯被放置在更大的深度。当相机运动（基线）较大时，这些高斯会被观察到，并且在图5（右列）中显示出合理的外观。图5的最后一列还揭示了我们方法的一个限制。它是一个确定性、回归性的结构和外观模型，因此在存在模糊性的情况下会产生模糊的渲染：当基线非常大、在遮挡区域或相机后退时。通过额外的损失（感知损失[96] 或对抗性损失[22]）可以减少模糊性。或者，我们的方法可以作为类似于[8]的框架中的条件，也可以作为基于扩散的前馈3D生成框架中的重建器[67, 71]。

<p align="center">
<img width="615" alt="image" src="https://github.com/user-attachments/assets/746a914f-6434-4bc3-8762-09de83071b17">
<\p>

表4：消融研究。不同设计选择的消融结果。

| 模型                                    | RE10k - 域内          | NYU - 跨域            | KITTI - 跨域          |
|---------------------------------------|----------------------|-----------------------|----------------------|
|                                       | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
| Flash3D                               | 24.93  | 0.833  | 0.160   | 25.09  | 0.775  | 0.182   | 21.96  | 0.826  | 0.132   |
| 没有深度网络，使用第二层               | 23.62  | 0.782  | 0.186   | 23.73  | 0.732  | 0.210   | -      | -      | -       |
| 没有深度网络，不使用第二层             | 24.01  | 0.806  | 0.176   | 23.98  | 0.750  | 0.207   | -      | -      | -       |
| 使用深度网络，不使用第二层             | 24.45  | 0.825  | 0.163   | 24.83  | 0.767  | 0.190   | 21.50  | 0.812  | 0.141   |
| 使用深度网络，仅反投影                 | 22.80  | 0.781  | 0.207   | 22.14  | 0.729  | 0.217   | 20.86  | 0.774  | 0.154   |

### 5 结论

我们提出了Flash3D，一个可以在单个GPU上仅用16小时训练即可实现单目场景重建的最新成果的模型。我们的公式允许使用单目深度估计器作为完整3D场景重建的基础。因此，该模型具有很好的泛化能力：即使未在目标数据集上专门训练，它也能优于以往的工作。分析揭示了预训练网络和学习模块之间的交互机制，消融研究验证了每个组件的重要性。

### A 数据集详情

#### RealEstate10k

我们从提供的链接下载视频，得到超过65,000个视频，以及提供的相机姿态轨迹。使用提供的相机，我们运行COLMAP [57]进行稀疏点云重建。我们使用MINE提供的测试拆分，并按照先前的工作在源帧前5帧和10帧的新帧上评估PSNR。此外，我们还在从±30帧间隔中随机采样的帧上进行评估。我们使用与[36]相同的帧进行评估。结果，我们在3205帧上进行了评估。我们使用其发布的检查点和常见的在图像边框周围裁剪5%的协议重现了[36]的结果，获得了与原论文中类似的分数。我们与BTS [85]的作者确认了这是常用的协议。我们的训练和测试分辨率为256 × 384。

#### NYUv2

我们形成了一个本质上类似于RealEstate10k的基准，它展示了室内场景，但在视觉上有很大的不同。我们下载了80个NYUv2 [61]的原始序列，并在其上运行COLMAP [57]以恢复相机姿态轨迹。在每个视频中，我们随机采样3个源帧，并从源帧的±30帧范围内均匀随机采样一个帧，反映了RealEstate10k的协议。我们对图像进行去畸变，并重新调整到256 × 384分辨率。

#### KITTI

我们在KITTI [18]数据集的Tulsiani测试拆分[76]上进行评估。KITTI数据集中的相机是以公制尺度提供的，我们的网络直接使用提供的相机和场景，无需额外的预处理。为了评估，按照先前的工作[36, 85]，我们裁剪图像外部的5%。

### B 基准和竞争方法

#### B.1 深度反投影

在我们的实验中，一个重要的基准是测量单目深度预测在单目新视图合成中的性能。在这个基准中，我们在单目深度预测器预测的深度位置放置具有固定不透明度的各向同性3D高斯，而没有视图依赖效应（即具有柔和边界的点云）。我们将高斯颜色设置为输入视图的缩放副本，即 

$$
c_G = \alpha c_{RGB}
$$ 

并初始化 

$$
\alpha = 1.0
$$ 

我们将高斯不透明度初始化为 

$$
\sigma = \text{sigmoid}(\sigma_0)
$$ 

其中 

$$
\sigma_0 = 4.0
$$ 

即几乎不透明。我们测试设置高斯尺度的两种变体：（1）高斯具有固定尺度 

$$
s = \exp(s_0)
$$ 

且 

$$
s_0 = -4.5
$$ 

（2）半径与相机的深度成比例，使高斯能够适应从像素投射的光线：

$$
s = \exp(s_0 \frac{d}{d_0})
$$ 

其中 

$$
d
$$ 

是UniDepth输出的公制深度，

$$
d_0 = 10.0
$$ 

且 

$$
s_0 = -4.5
$$ 

接下来，虽然我们确定了 

$$
\alpha = 1.0
$$ 

$$
s_0 = -4.5
$$ 

和 

$$
\sigma_0 = 4.0
$$ 

是合理的初始化，但它们可能不对应最高质量的新视图合成。因此，我们对该基准的参数进行基于梯度的优化，优化 

$$
\alpha
$$ 

$$
s_0
$$ 

$$
\sigma_0
$$ 

以最小化训练集中的源视图和3个新视图（与我们的最终模型相同）中的光度损失。我们训练这些模型进行X次迭代，并选择在验证拆分上表现最佳的模型。最后，我们在测试拆分上评估最佳 

$$
\alpha
$$ 

$$
s_0
$$ 

$$
\sigma_0
$$ 

的模型并报告度量结果。

表5：深度反投影基线。我们通过基于梯度的优化拟合深度反投影模型的超参数。我们尝试了两种变体：一种是具有固定大小的高斯，另一种是高斯尺度随着深度成比例增加。前两行是在修正深度反投影从像素中心而不是像素角之前的结果。所有度量都使用裁剪测量。

| 模型                | 骨干网络    | 5 帧                | 10 帧               | 随机帧            |
|---------------------|------------|---------------------|---------------------|-------------------|
|                     |            | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
| 固定大小             | ConvNeXT-L | 26.47  | 0.864  | 0.120   | 24.08  | 0.808  | 0.173   | 22.60  | 0.774  | 0.211   |
| 固定大小             | ViT-L      | 26.62  | 0.867  | 0.120   | 24.25  | 0.814  | 0.172   | 22.78  | 0.781  | 0.209   |
| 深度依赖             | ConvNeXT-L | 26.49  | 0.861  | 0.124   | 24.10  | 0.806  | 0.175   | 22.61  | 0.774  | 0.209   |
| 深度依赖             | ViT-L      | 26.65  | 0.864  | 0.123   | 24.29  | 0.812  | 0.173   | 22.80  | 0.781  | 0.207   |

### B.2 Splatter Image

我们使用与我们自己方法相同的基于ResNet-50的U-Net卷积神经网络实现了Splatter Image基线，以进行公平的比较。我们在两块NVIDIA A6000 GPU上训练了350,000步，比我们提出的Flash3D多一个数量级。训练花费了6个GPU天，与[68]中报告的相同。

### B.3 MINE

MINE [36] 只提供了模型权重，但没有在RealEstate10K数据集上的推理和评估代码，因此我们重新运行了推理和评估以确保可重复性。结果与[36]中报告的相符。我们使用N = 64的模型，因为这是作者提供的最佳模型。对于NYU的评估，我们使用在Re10k上训练的模型，与我们的方法相同。

### B.4 双视图方法

在与双视图方法比较时，我们需要选择其中一个作为我们的源视图。对于任何方法，目标帧性能的最显著因素是与源帧的基线。我们在256×256分辨率下进行此比较，而不进行边框裁剪，以实现完全可比性。

### B.5 高斯概率分布

多高斯的另一种方法是像pixelSplat [9]中那样预测深度概率。然而，如果没有预训练深度预测器的估计深度，覆盖速度非常慢，并且在我们的单目设置中性能更差。为了公平比较，我们仅对其他高斯层进行消融研究，即K > 1的高斯。结果在表6中报告。连续的深度偏移优于pixelSplat中的深度概率设计。

表6：深度解码器架构的消融研究。这里，我们消融了类似于pixelSplat [9] 中的概率深度，但仅对每像素 \( K > 1 \) 的高斯进行。这里，跨域 (CD) 表示该方法未在正在评估的数据集上进行训练。

| 方法                    | CD | KITTI                     | NYU                      |
|-------------------------|----|---------------------------|--------------------------|
|                         |    | PSNR ↑ | SSIM ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
| Flash3D (Discrete)-2    | ✓  | 21.35  | 0.805  | 0.153    | 24.52  | 0.763  | 0.200   |
| Flash3D (Discrete)-3    | ✓  | 21.50  | 0.814  | 0.136    | 24.84  | 0.772  | 0.189   |
| Flash3D (Ours)-2        | ✓  | 21.96  | 0.826  | 0.132    | 25.09  | 0.775  | 0.182   |

### C 实现细节

#### C.1 架构

我们基于ResNet-50 [24] 骨干网络并实现了一个U-Net [54] 编码-解码器，如[21]中所述。具体来说，单个ResNet编码器由多个解码器共享，每个解码器用于外观参数的每一层以及深度偏移解码器，除了第一层的偏移解码器，因为我们直接从预训练模型中获取深度值。

#### C.2 优化

我们定义光度损失如下[20]，作为

$$
L_1
$$

和

$$
\text{SSIM} [81]
$$

项的加权和：

$$
\mathcal{L} = \| \hat{J} - J \| + \alpha \, \text{SSIM}(\hat{J}, J)
$$

与之前的工作[36, 74]不同，我们不使用稀疏深度监督。

其中

$$
J
$$

是目标图像，

$$
\hat{J}
$$

是渲染，且

$$
\alpha = 0.85 。
$$

我们使用Adam [32] 优化器以批次大小为16和学习率为0.0001的设置对网络进行优化，共进行40,000次训练步骤。

#### C.3 尺度对齐

相机姿态通常使用COLMAP估计。这些相机姿态在每个场景中的尺度是任意的。按照之前的工作，我们使用[74]中的尺度因子计算，将COLMAP相机的尺度对齐到我们网络估计的尺度。然而，如果深度估计中存在离群值（无论是在我们的方法中还是在基准中），它们将影响尺度估计。结果，场景重建的尺度与渲染新视图的相机姿态的尺度之间可能存在不匹配。因此，渲染的新视图可能会相对于真实值发生偏移，这不会显著影响LPIPS，但会影响PSNR。因此，在测试时，我们使用RANSAC进行尺度对齐。在评估MINE时，我们也在传输数据集NYU上执行相同的操作，因为其深度预测的准确性在这个未见过的数据集中会恶化。在估计尺度时，我们使用RANSAC方案，样本大小为5，迭代次数为1,000，阈值为0.1。

### D 局限性

提出的方法的主要局限性在于它是一个确定性、回归模型。这激励它在存在不确定性的情况下生成模糊的渲染，例如当基线非常大时，在被遮挡区域或相机向后移动时。

另一个局限性是重建器无法捕捉所有被遮挡的表面：重建的3D模型仍然有一些洞。虽然许多这些区域被填补了，但有些被遗漏了，即使预测了多个高斯。

最后，预训练深度估计器的失败可能导致我们的场景重建失败，特别是当估计的深度被高估时。这是由于我们的深度偏移是非负的，因此无法恢复比预训练深度估计器估计的表面更接近相机的场景结构。这使得模型在推理时依赖于第三方模型在使用域中的质量。

### E 更广泛的影响

本研究关于单目场景重建，具有潜在的正面和负面社会影响。

从正面来看，该方法显著减少了获取野外3D资产所需的计算和时间资源，为具有积极影响的消费应用开辟了道路。例如，快速重建房屋以促进其销售；数字化保存文物和文化遗址；以及在安全自动驾驶中的应用。

从负面来看，这项技术有可能被用于恶意目的，例如非法或不道德的跟踪和监视，或侵犯个人隐私，例如未经同意重建他人的身体。此外，如果用于自动驾驶和机器人等应用中，错误的预测可能会造成伤害，因为误判的3D结构可能导致碰撞或性能不佳。
