什么是VIT模型？
**原始的 Vision Transformer（ViT）本身是判别模型，不是生成模型**。
## 标准 ViT：不能直接生成图片

最早的 ViT 设计目标是：

`图像 → token → Transformer → 分类结果`

它只学习：

- 如何理解图像
    
- 如何提取特征
    

没有：

- 解码器（decoder）
    
- 上采样模块
    
- 像素重建头

目前主流的生成方式是：扩散模型 + ViT Backbone：
- DiT（Diffusion Transformer）
    
- U-ViT
    
- PixArt-α 等