# 📖 model-deploy · 中文小说模型测试（AMD ROCm 主路线）

在 ModelScope **免费 AMD GPU 实例**（192GB 显存 · ROCm）上测试中文小说模型。当前分支（`amd-xuanhuan-bf16`）的第一目标：**玄幻 DPO（JiangLing-js/Qwen3.8-27B-Chinese-Xuanhuan-Novel-Writer-DPO）**，以最接近作者训练条件的方式运行——**BF16（未量化）底模 + LoRA**，测的是模型本身，不是"模型+4bit"。

**你不需要懂 Linux、Python、CUDA 或任何部署知识。**

| 测试对象 | 运行方式 | 首次下载 | 当前状态 |
|---|---|---|---|
| ② 玄幻 DPO（第一优先） | Qwen3.8-27B **BF16** + LoRA · Transformers/PEFT | ≈56.6GB | ✅ 本分支主路线（AMD ROCm 192GB） |
| ① WebNovel Writer | llama.cpp + Q4_K_M | ≈16.5GB | NVIDIA 历史路线（见文末），AMD 适配属第二阶段 |

> 本项目只做小规模推理测试，不包含训练/微调/Web 服务/API/Docker。与 wcn123、JiangLing-js、unsloth、ModelScope 均无隶属关系。

---

## 🚀 完整使用流程（七步）

### 第 1 步 · 注册 ModelScope（约 5 分钟）

打开 [https://www.modelscope.cn](https://www.modelscope.cn) 注册（手机号）并完成**实名认证**（免费 GPU 需要）。AMD GPU 免费额度约 **100 小时**，按开机时间计；CPU 实例免费不限时。

### 第 2 步 · 打开本分支的 Notebook 映射链接

浏览器打开（已按当前分支填好）：

```
https://modelscope.cn/notebook/share/github/florent0921-lgtm/model-deploy/blob/amd-xuanhuan-bf16/modelscope_test.ipynb
```

点右上角 **「在 Notebook 中打开」**。（此分支验证通过后会合并回 main，届时可把链接里的 `amd-xuanhuan-bf16` 换成 `main`。）

> 打不开映射链接时的备用方法：在「我的Notebook」手动创建实例（选第 3 步的 AMD 镜像），把仓库里的 `modelscope_test.ipynb` 下载后直接拖进上传，右键打开。

### 第 3 步 · 选 AMD GPU 实例并启动

创建环境时：

- **镜像**：选 **`ubuntu22.04-rocm7.2.3-py312-torch2.11.0-1.39.0`**（ROCm + PyTorch 平台环境，平台已适配好，**不要自己动 PyTorch**）；
- **GPU**：选 **AMD GPU（192GB 显存）** 实例；
- 点**启动**，等 2~5 分钟进入 Notebook。

### 第 4 步 · 运行全部

打开 `modelscope_test.ipynb` → **Run → Run All Cells**：

- 环境自检会识别出 AMD ROCm GPU（显示 GPU 名称、显存、ROCm 版本）；
- 第 5 步自动下载并加载模型②：**BF16 底模 55.6GB + LoRA 约 1GB**（首次 20~60 分钟；中断重跑会**断点续传**；下载后永久保存在 100GB 持久盘，以后开机只需 5~15 分钟从磁盘读入显存）；
- 如果 Hugging Face 官方源连不上，自动切第三方镜像 hf-mirror.com，无需操作；
- 加载完成后**自动跑 Smoke Test**（极短自检生成）：验证 AMD GPU、BF16 驻留显存、思考模式关闭、输出正常——看到
  **“✅ 玄幻 DPO 已在 AMD ROCm GPU 上准备完成”** 就绪。

### 第 5 步 · 粘贴剧情提示词，生成正文

第 6 步输入框已预填**作者官方 〖〗结构示例**（人物状态/前情/事件骨架/写作要求）：

1. 把示例换成你的剧情（保留 〖〗结构只改内容效果最好；直接粘贴纯骨架也行，程序会自动套结构）；
2. 点 **【✍️ 开始生成】**，正文逐字显示（约 1~5 分钟，目标 500~1000 字）。

生成参数为作者 held-out 测试原参数：temperature=0.85 / top_p=0.90 / top_k=40 / 重复惩罚=1.05 / 上限 1700 token / 思考模式关闭。程序还会自动检查 Prompt 长度：超过 2800 tokens（作者训练分布的提示词上限）会拦截并提示缩短，不会自动截断你的剧情。

### 第 6 步 · 查看与保存结果

正文显示的同时**自动保存**到 `/mnt/workspace/novel-model-benchmark/results/`：

- `xuanhuan-dpo_日期时间_测试编号.txt` —— 纯正文；
- 同名 `.md` —— 附模型、**运行环境**（GPU/ROCm/torch/transformers 版本）、精度（BF16）、参数、耗时、显存峰值、提示词全文；
- 下载到本地：左侧文件列表 → 右键 → **Download**。

### 第 7 步 · 关闭 GPU（省时长）

ModelScope 网页 →「我的Notebook」→ 实例 → **停止**。额度只按开机时间扣；`/mnt/workspace` 里的模型和结果全部保留；只有**删除实例**才清空。

---

## 💡 省时长技巧（可选）

GPU 时长按开机时间计（下载也在计时）。想省额度：先在**免费不限时的 CPU 实例**上 Run All——Notebook 会自动识别无 GPU 环境并进入「预下载模式」（只下载模型②的 56.6GB，不加载不生成），完成后停止 CPU 实例、再开 AMD 实例直接加载。

## 🧪 批量盲评模式（可选）

把测试剧情放进 `prompts/`（一个 `.txt` 一条，示例自动生成），把第 8 步的 `BATCH_MODELS` 改成 `["xuanhuan"]` 重跑该 Cell。匿名结果在 `results/anonymous/`，对照表「AAA_评分后再看_对照表.txt」评分前别打开。模型①适配 AMD 后可恢复双模型盲评。

---

## ❓ 常见问题

**Q1：为什么不用 4bit？显存不是够吗？**
就是要用"显存够"来换"测试干净"：第一轮要判断的是 AI 味、词语搭配这类细微差别，不能引入量化这个新变量——否则出了怪表达分不清是模型的问题还是 4bit 精度损失。192GB 下 BF16（约 56GB）绰绰有余，模型卡作者的官方用法也是不量化推理。

**Q2：下载模型也扣额度吗？**
会（按开机时间计）。用上面的 CPU 预下载技巧可以让下载完全不占 GPU 时长。100 小时对推理测试非常充裕。

**Q3：下载中断/网络报错？**
重新 Run All 即可，断点续传从断点继续；程序会自动在官方源和第三方镜像 hf-mirror.com（非官方节点，仅下载公共文件、不涉及令牌）之间切换。

**Q4：磁盘要多大？**
BF16 底模 55.6GB + LoRA + 缓存余量，建议 `/mnt/workspace` 剩余 ≥70GB（Notebook 下载前会自动检查）。如果看到"旧的 4bit 底模缓存（约22GB）"提示，确认不需要 A10 历史路线后可按提示手动清理。

**Q5：Smoke Test 失败怎么办？**
说明环境或兼容性问题（不是你的操作错误）。把 ❌ 条目截图求助，正式测试已被程序自动阻止，不会产生失真的对比结论。

---

## 🗂 分支说明

| 分支 | 内容 |
|---|---|
| `amd-xuanhuan-bf16` | **当前主推**：AMD ROCm 192GB + BF16 + LoRA + Smoke Test（本 README） |
| `main` | 历史方案：NVIDIA A10 24GB（WebNovel GGUF + Xuanhuan 4bit + OOM 预案）——AMD 版验证通过后 main 将更新 |

A10 历史路线为**设计方案，尚未在本轮测试中实际 GPU 验证**（本机无 NVIDIA 显卡，未真实跑通）：在 NVIDIA A10 实例上打开 main 分支的 Notebook 即可尝试（模型① llama.cpp 路线，模型② 4bit 属尝试性支持设计）。

## 📜 许可与用途（重要）

本项目及其生成结果**仅供个人对比测试与研究使用**。

- **玄幻 DPO（模型②）**：许可标注 `other`，作者明确说明训练语料包含**受版权保护文学作品的派生材料**，适配器不授予对底层文学作品的任何权利。仅用于本次个人测试；任何正式或商业接入前需单独审查许可与训练语料版权风险。
- **WebNovel Writer（模型①）**：遵循 Qwen License，模型卡注明"仅供研究与个人使用"。商用前请重新核对许可与训练数据权利。

生成内容为 AI 产出，请自行判断并承担使用责任。

## 🗂 项目结构

```
model-deploy/
├── modelscope_test.ipynb   # 唯一需要的东西：全部流程都在这个 Notebook 里
├── prompts/                # 批量盲评用的测试剧情（可自行替换）
│   ├── 01_寒潭.txt
│   ├── 02_战斗.txt
│   └── 03_突破.txt
├── results/                # 生成结果（在云端 /mnt/workspace 自动创建，不入库）
└── README.md
```
