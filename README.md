# Flash3D: Feed-Forward Generalisable 3D Scene Reconstruction from a Single Image
Stanislaw Szymanowicz*, Eldar Insafutdinov*, Chuanxia Zheng*, Dylan Campbell, João Henriques, Christian Rupprecht, Andrea Vedaldi
- Visual Geometry Group - University of Oxford

## 摘要

本文提出了一种名为Flash3D的方法，用于从单张图像进行3D场景重建和新视角合成。该方法既具有很高的泛化性，又非常高效。

在泛化性方面，我们从单目深度估计的“基础”模型出发，将其扩展为一个完整的3D形状和外观重建器。在效率方面，我们基于前馈高斯点积来进行扩展。具体来说，我们在预测的深度处预测第一层3D高斯，然后在空间中偏移添加额外的高斯层，从而使模型能够在遮挡和截断后完成重建。

Flash3D非常高效，可以在单个GPU上在一天内完成训练，因此大多数研究人员都可以使用。

在RealEstate10k数据集上进行训练和测试时，Flash3D达到了最先进的结果。当迁移到NYU等未见过的数据集时，它以较大优势超过了竞争对手。更令人印象深刻的是，当迁移到KITTI时，Flash3D在PSNR上比专门在该数据集上训练的方法表现更好。在某些情况下，它甚至超过了使用多视图输入的最新方法。



