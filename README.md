# GPU 开源生态挑战赛

## 📌 挑战方向

### 模型迁移
- 单 Transformers 模型迁移到 vLLM / SGLang（可运行并优化）
- 参考LMDeploy开源适配模型实现，实现未支持的多型在国产算力上运行
- 其它有价值的迁移，以最终提交审核为准

### 论文复现
- AI4S 领域前沿论文复现与性能加速

### 生态破局
- 将 CUDA AI 开源软件迁移至国产开源MACA软件栈

### 生态繁荣
- Jax 模型迁移至 PyTorch生态
- 参与MinerU开源社区上游贡献支持MACA软件栈，例如：https://gitee.com/ccf-ai-infra/GPUApps/issues/ICUVB2
- 参与InfiniCore开源项目的上游贡献支持MACA软件栈，例如项目主仓库：https://github.com/InfiniTensor/InfiniCore, 参考的迁移方式见主仓库的其他算子迁移到metax的方式以及：https://github.com/wawahejun/InfiniCore-Intern, 项目文档与概况详见：[DEV.md](./docs/InfiniCore-Documentation/DEV.md)
- 参与github.com/MetaX-MACA组织的开源项目上游贡献，例如：https://github.com/MetaX-MACA/vLLM-metax/issues/7

### 可参考项目列表
- https://github.com/MetaX-MACA/FlashMLA

## 其它方向
▶ 项目价值：推动国产 AI 基础设施自主化的有价值的领域

## 📤 提交要求及评分标准

### 参赛资格
- 提交的代码可在MACA软件栈+曦云C500上运行的内容。

### 提交内容（提交到比赛平台、当前仓库或者上游开源社区的代码仓库提供社区PR Link到对应的赛题Iusse中）
- 代码
- 部署脚本
- 环境配置
- 在线复现环境
- 文档

### 文档要求
- 验收测试用例
- 使用手册

### 评分标准
- 能成功复现运行：60 分
- 输出正确结果（参考 CUDA 平台）：61~100 分

### 加分项
- 提供性能优化，且提供可复现的对比测试报告：+0~50 分

## 🏆 排名机制
- 评委评分从高到低排序
- 若分数相同：
  - 性能提升高者优先
  - 内容完整性高者优先