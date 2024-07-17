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

总体而言，Flash3D是一个简单且高性能的单目场景重建管道。经验表明，我们发现Flash3D可以(a)渲染高质量的重建3D场景图像；(b)在广泛的场景中操作，包括室内和室外；并且(c)重建被遮挡部分，这在仅深度估计或天真的扩展中是不可能实现的。Flash3D在所有指标上在RealEstate10K\[72\]上实现了最先进的新视图合成精度。
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

## 3 方法



$$
设I \in \mathbb{R}^{3 \times H \times W}是一个RGB图像。我们的目标是学习一个神经网络 
$$



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
\hat{J}
$$

。

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
\sigma
$$

，深度 

$$
d \in \mathbb{R}_+
$$

，偏移 

$$
\Delta \in \mathbb{R}^3
$$

，协方差 

$$
\Sigma \in \mathbb{R}^{3 \times 3}
$$

表示旋转和缩放（使用四元数表示旋转），以及颜色模型 

$$
c \in \mathbb{R}^{3(L+1)^2}
$$

，其中 

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
\Psi
$$

。
