# 面向多模态离散扩散模型的生成优化理论与方法

## 一、国内外研究进展、现状、挑战与意义

### 1.1 研究背景、问题范围与选题意义

近年来，多模态模型的任务边界已从分类和检索扩展到理解、生成与交互。以同一主干处理文本和视觉输入，可以同时支持视觉问答、文生图、图像编辑和多轮修正。更值得关注的是，理解分支获得的对象、属性和关系知识可以反过来约束生成，生成结果也可以再次被模型读取和修正。这种能力为自反馈优化提供了基础，也适用于内容创作、视觉设计等需要反复修改的场景。

文生图为观察多模态模型的生成能力提供了直观入口。尽管现有模型已能生成感知质量较高的图像，面对包含多个对象及复杂关系的提示时，仍常出现对象遗漏、计数错误、属性错配和空间关系颠倒。GenEval 与 DPG-Bench 分别从对象组合和稠密提示遵循两个方面记录了这些问题[1–2]。近期研究发现，重新分配训练信用或推理预算可以纠正其中一部分错误[3–5]。因此，在底座模型和计算预算基本固定时研究奖励传递与计算分配，是继续扩大模型规模之外另一条值得研究的路径。

本课题聚焦一类具体模型：它使用离散视觉 token 表示图像，通过多轮掩码预测完成生成，并以共享主干承担视觉理解任务。与连续潜空间扩散、自回归视觉 token 模型以及“语言模型加独立连续生成器”的组合系统相比，这类模型的生成顺序并不固定，同一图像可以由不同的去掩码路径得到。由此产生的轨迹概率、信用分配和跨步学习问题，构成本课题的主要研究范围。实证工作以文本条件图像生成和文本—图像理解为主，不把结论直接外推到语音、视频等其他模态。

下文沿生成轨迹、训练信用、推理搜索和少步迁移梳理相关研究。非固定去掩码顺序使生成秩、可见上下文和终局奖励之间不再具有固定对应关系；统一模型可以读取自己的输出，又使内部反馈成为可能。由此，本课题关心怎样判断哪些生成决策更重要，并把有限的训练与推理计算投入这些决策。最终收益以生成质量、理解能力和实际运行成本共同衡量。

### 1.2 图像生成范式、离散视觉表示与掩码扩散

扩散模型先逐步扰动数据，再学习反向恢复过程。DDPM 证明了多步去噪的有效性[6]；潜空间扩散把去噪转移到压缩潜变量中，降低了高分辨率生成成本[7]；SDXL 则通过更大的网络、条件编码和训练策略改善高分辨率合成[8]。连续表示便于描述细粒度变化，采样理论和工程工具也较成熟。渐进蒸馏、一致性模型和分布匹配蒸馏进一步表明，多步采样可以压缩到少数步甚至一步[9–11]。

离散图像生成首先要把像素压缩为有限词表。VQ-VAE 奠定了神经离散表示的基本形式[12]，VQGAN 又结合感知损失和对抗训练改善重建质量[13]，DALL-E 展示了把文本与离散图像 token 放入同一序列建模的可行性[14]。这类方法让 Transformer 可以用相近的接口处理文本和图像，但生成上限也受到 tokenizer 约束：保留更多视觉 token 有助于保存纹理、文字和小物体，却会增加序列长度；更强压缩降低主干计算量，也可能在建模前就丢失细节。因此，评价系统时还应报告视觉 token 数、码本利用率和重建质量。

早期离散图像模型多按预定顺序逐 token 生成，概率分解清楚，但图像序列较长，解码延迟也较高。MaskGIT 改用双向 Transformer 反复预测被掩码位置，每轮保留高置信 token 并重新掩码低置信位置，从而并行恢复多个位置[15]。D3PM 从离散状态空间的马尔可夫转移出发建立更一般的离散扩散框架[16]，后续比率估计方法又扩展了训练与采样形式[17]。因此，离散图像生成并不限于自回归，掩码调度也可以改变恢复顺序和采样步数。

设视觉 token 序列为 z=(z1,…,zN)，初始状态全部为掩码 [M]。第 t 轮中，模型根据文本条件 c 和当前可见 token 预测其余位置的类别分布，再依据置信度、随机采样和掩码日程决定本轮写入哪些位置。经过 T 轮迭代，完整 token 序列由图像解码器还原。这里真正困难的并非把类别预测称为动作，而是定义整条生成轨迹：并行位置选择会改变轨迹概率，同一终态对应多条去掩码路径，随机重掩码还可能产生真实采样未访问过的上下文。

二维图像展平后仍需保留空间关系。模型通常借助二维位置编码、行列标记或分块机制补充空间结构。每轮恢复更多位置可以减少同步次数，却也可能过早固定错误结构；每轮只恢复少量位置则会增加采样轮数。因而，并行预测 token 只是影响延迟的一个因素，实际质量和速度还取决于 tokenizer、空间编码、掩码调度及每轮主干计算量。

《Sketch Then Paint》在其底座和采样协议下，把较早恢复且预测熵较高的 token 作为结构 token 的近似，并观察到层次更新带来的收益[3]。该结果说明生成顺序可以作为结构先验，但还不能据此认定所有早期 token 都对应轮廓或布局。例如，背景 token 可能先稳定，而复杂主体的边界可能较晚才确定。后续方法还需结合预测熵、空间位置和终局奖励敏感度检验这一先验。

### 1.3 生成与理解统一模型的技术谱系及比较

本文把多模态模型的“统一”区分为主干共享、表示共享和训练目标共享。一个模型同时支持问答和文生图，并不表示三个层面都已经统一。按图像生成机制，现有工作大致有三条路线：Emu3 把文本和离散图像都作为自回归序列[18]；Janus-Pro 使用不同视觉编码路径处理理解与生成，再共享主体模型[19]；Show-o 在同一 Transformer 内对文本采用自回归目标、对图像采用离散扩散目标[20]。MMaDA、Muddit、Lumina-DiMOO 和 LaViDa-O 则进一步用双向注意力与掩码预测处理多模态 token，探索统一的离散扩散目标[4, 21–23]。

语言扩散模型为后一条路线提供了初始化和训练基础。LLaDA 将掩码扩散扩展到语言建模[24]，从自回归模型迁移参数的研究表明，这种适配可以降低从头训练的成本[25]。LLaDA-V 又通过视觉指令微调使扩散语言模型读取图像并回答问题[26]，但视觉理解本身不等于具备图像生成能力。现有论文往往同时改变 tokenizer、模型规模、训练数据和训练目标，因此仅凭公开结果很难判断增益来自哪一部分，不同路线仍需在相近底座和评测协议下比较。

所附 Lumina-DiMOO 论文给出了全离散统一模型的完整实例[4]。该模型以扩散语言模型为初始化，在文本词表之外使用包含 8192 个离散符号的视觉词表，使用同一个双向 Transformer 和掩码预测目标处理两种模态。其图像 tokenizer 的空间下采样率为 16，1024×1024 图像对应 4096 个视觉位置，并通过行结束标记保留二维形状信息。训练由图文预训练、多任务中期训练、监督微调和 Self-GRPO 组成，覆盖文生图、编辑、主体驱动、控制生成、补绘、外扩和视觉问答。这个例子说明架构统一需要数据和阶段化训练配合；仅共享主干并不能自动消除生成与理解之间的目标冲突。

在论文报告的同一设置内，Lumina-DiMOO 的 GenEval 总分由 0.88 提升到 0.91，颜色和属性绑定分项也有改善[4]。这些结果说明理解分支提供的反馈可以改善生成，但 GenEval 只评价生成图像是否符合对象和属性要求，不能证明理解能力也随之提高。要支持“生成—理解协同”，还需验证独立理解基准不退化，并确认内部评价与人工或外部判断一致。

统一模型还面临长序列和多轮采样的系统矛盾。Lumina-DiMOO 在所报设置下使用最大 logit 缓存复用稳定 token 表示，将 1024×1024 文生图时间由约 45 秒降至约 22 秒，同时峰值显存由 38.9 GB 增至 45.9 GB，且缓存为有损近似[4]。这一结果体现了计算、内存和误差的交换：网络级缓存减少重复计算，但不决定哪些候选值得继续；序列剪枝可以缩短输入，却可能破坏全局关系；算法级预算分配能够减少低价值轨迹或步骤，却要承担评分与调度开销。后续工作需要在相同硬件、批量、分辨率与解码设置下分别报告主模型、验证和图像解码成本，才能比较不同层面的效率优化。

三类统一模型各有取舍：自回归方法的概率分解清楚，但图像解码较慢；混合目标方法保留了各模态较成熟的训练方式；全掩码扩散支持双向条件建模和并行恢复。本课题选择第三类，是因为去掩码顺序、内部反馈和采样步数都可以进一步优化。这一选择服务于本课题的问题设置，不代表其他路线在能力或效率上普遍较弱。

### 1.4 奖励后训练、离散策略概率与层次信用分配

掩码预测训练直接优化局部 token 的预测，而实际评价看的是整幅图像的提示遵循、感知质量和人类偏好。为缩小两者的差距，研究者开始用整图奖励继续优化生成模型。PPO 通过裁剪概率比限制策略更新[27]，GRPO 利用同一输入下多条样本的相对奖励估计优势[28]。在连续扩散中，DDPO 把逐步去噪视为策略轨迹[29]，Diffusion-DPO 则把成对偏好转化为直接偏好目标[30]。这些方法说明终局奖励可以优化扩散生成，但没有直接回答离散掩码过程中“哪些 token 在何时构成动作”。

离散掩码生成的困难首先来自轨迹定义。模型在一轮内同时预测多个位置，再由置信度、随机性和掩码日程决定写入集合。同一图像可以经由不同顺序生成，因此轨迹概率既包含类别预测，也与位置选择有关。若把最终 token 彼此独立地处理，会遗漏并行选择和调度的影响；若保存每轮状态及新旧策略概率，内存与计算又会随序列长度、采样步数和采样数量增长[3, 31–32]。离散扩散强化学习需要在轨迹真实性、概率估计偏差和存储开销之间作出取舍。

离散扩散强化学习常见三种训练上下文。MaskGRPO 从最终样本随机重掩码，不必保存完整轨迹，但可见 token 可能包含真实生成时尚未出现的信息[31]。

另一种做法保存实际去掩码轨迹。TraceRL 主要研究扩散语言模型，《Sketch Then Paint》据此设置了图像生成对照 TraceGRPO[3, 32]。这种上下文更接近行为轨迹，代价是缓存长序列中间状态。

全掩码代理只保留文本条件，计算较规整，也彻底删去了真实过去信息[3]。因此，三种做法应同时比较概率偏差、存储开销和最终性能。

第二个困难是把图像级奖励分配给 token 级决策。把同一优势广播给所有视觉 token，相当于假设各位置和阶段贡献相同；固定提高早期 token 的权重也不等于因果归因，因为同一部分结构经过不同补全可能得到不同结果。这里需要区分状态价值和动作边际贡献：前者表示从当前状态继续生成的预期回报，后者考察替换或重新掩码某个 token 后回报如何变化。后者通常需要多次条件补全或反事实采样，计算较贵，因此现有方法多用生成秩、预测熵或分组权重近似。

《Sketch Then Paint》提出 HT-GRPO，将 rollout 中较早恢复且高熵的视觉 token 视为结构集合，较晚 token 视为细化集合，在全局、结构和细化三个阶段分配不同训练预算与信用权重[3]。该方法使用全掩码文本条件估计概率比，以避免随机重掩码的未来信息污染和真实轨迹的缓存开销。其代价是代理分布不含真实已生成上下文，不能与行为轨迹条件概率等同。方法所利用的“早期更偏结构”应理解为统计先验，而不是逐 token 的语义标注。

在该稿件所报的同一实验协议内，HT-GRPO 将 MMaDA 的 GenEval 从 0.56 提升至 0.83，MaskGRPO 对照为 0.80；在 Lumina-DiMOO 上由 0.83 提升至 0.92，MaskGRPO 对照为 0.88，DPG-Bench 也报告同方向增益[3]。这些结果支持层次信用在指定底座、提示集与评价器上的有效性，但尚不能证明固定结构比例或全掩码概率代理具有普适性。稿件属于最新公开工作，相关结论还需在更多采样日程、分辨率和非文生图任务中验证，并与相同训练预算、rollout 数量和奖励模型的基线比较。

现有工作已经证明终局奖励可以用于扩散生成优化[29–30]，并提出了面向离散轨迹的概率代理和层次信用方法[3, 31–32]。目前仍缺少对代理偏差、轨迹存储成本和动态信用的统一比较。除最终生成指标外，实验还应报告策略 KL、裁剪率、多样性、训练显存和 GPU 时间，以判断提升是否来自奖励过拟合、模式收缩或额外计算。

### 1.5 生成—理解自反馈与推理时轨迹扩展

统一模型能够读取自己生成的图像，因此可以把理解分支用作内部评价器。CLIP 和 ImageReward 等外部评价器与生成模型相互独立，评价标准相对稳定，但会增加计算成本，也可能与目标数据或人工偏好不一致[33–34]。内部评价器可以把提示拆成对象、数量、颜色和关系问题，再由视觉问答分支逐项检查。它的主要风险是生成器与评价器可能共享同一盲区，因此模型自洽并不表示判断正确。

UniRL 研究了统一多模态模型的监督与强化学习自改进[35]。Lumina-DiMOO 的 Self-GRPO 则让同一提示生成多张图像，把实体、关系、数量和颜色转成选择题，再用理解分支的答题结果构造组内奖励[4]。这种反馈能够区分计数、绑定和关系错误，并复用统一主干。现有结果已经说明它对生成有效，理解能力是否同步保持仍需单独测量。

内部评价器可能绕过图像，只根据提示先验猜答案，因此需要无图、换图、遮挡和反事实问题对照。若生成与评价分支同步更新，评分标准还会随策略变化；冻结评价器虽能稳定标准，却可能不再适应新的生成分布。此外，结构化问答适合检查对象和属性，但难以覆盖审美、文字、局部伪影和多样性。因而，内部评分还需与独立指标和人工盲评核对，并报告校准误差。

在参数保持不变时，另一种提高生成质量的途径是增加推理计算，即 test-time scaling。连续扩散中的代表工作从扩展初始噪声候选、以奖励或价值函数筛选、在中间状态搜索以及反思式迭代等方向展开[36–39]。最简单的 best-of-N 为同一提示完整生成 N 张图像再选择最好结果，容易实现和并行，但主模型成本随 N 近似线性增长；若每条候选又尝试 T 种细化设置，朴素组合需要约 N×T 组生成。树或分层搜索在中间状态剪枝，把预算集中给高潜力分支，却面临不完整图像难以评价的问题：越早剪枝越能节省计算，评价噪声和误删风险也越大。

dMLLM-TTS 将离散扩散的推理扩展分为轨迹探索和迭代细化两条轴[5]。轨迹探索扩大初始结构假设，迭代细化增加保留轨迹的恢复深度；其自验证把文本—图像一致性转化为视觉问答，并用模型自身理解分支评分。分层轨迹搜索先探索多个结构，再保留固定宽度 K 的候选继续细化，避免对所有 N 个候选执行全部 T 种后续配置。在 K 视为固定且只统计主模型前向的预算模型下，所需 NFE 可由朴素组合的量级 O(NT) 降为近似由探索与细化相加的 O(N+T)。这一表述依赖固定保留宽度、共享计算和具体实现，验证问题生成、问答、解码、数据搬运与不规则批处理并未包含在该渐近记号中。

CVPR 版本在 Lumina-DiMOO、MMaDA 和 Muddit 上报告了质量增益，并在其设定下相对线性搜索获得最高约 6 倍的主模型前向计算效率改善[5]。该最高值仅代表论文特定配置下的主模型计算变化，不等同于普遍端到端加速。扩展版 dMLLM-TTS v2 又加入多种采样调度器，并聚合提示遵循、审美和质量信号[40]。该稿件在 T=32、N=32、K=8 的代表设置下报告：线性轨迹搜索 GenEval 为 0.91，分层搜索为 0.93；在相近总体得分下，后者 NFE 约为前者五分之一。[40] 尚未形成正式卷期，本文仅将这些数字作为未发表作者稿在指定配置下报告的阶段性证据，不与正式发表结果合并比较。

无论用于训练还是搜索，效果都取决于评分是否可靠。训练中的错误反馈会持续改变参数，搜索中的错误评分则会提前淘汰仍可恢复的轨迹。完整图像适合评价细节和审美，中间状态主要用于判断对象和布局，评分方式应随生成阶段调整。多评分器能够覆盖更多维度，其额外计算也必须计入总成本。

现有证据已经表明，理解分支可以参与奖励构造和轨迹选择[4–5, 35, 40]。下一步需要回答的是：内部评分在不完整图像上是否可靠，怎样用独立评价识别共享偏差，以及搜索中的额外成本能否通过蒸馏转化为固定预算下的能力。

### 1.6 少步生成、扩散蒸馏与搜索经验迁移

蒸馏希望把多步教师的能力迁移给少步学生。渐进蒸馏让学生用一步逼近教师的两步更新[9]；一致性模型要求同一概率流轨迹上的不同状态映射到一致结果[10]；分布匹配蒸馏则利用真实与生成分布的得分差异训练一步生成器[11]。三类方法分别从局部轨迹、跨时间一致性和输出分布三个角度压缩采样过程，采用的监督信号并不相同。

这些方法不能直接照搬到掩码离散扩散中。连续状态之间可以计算数值距离，而离散学生的一步输出可能同时改变大量 token，教师预测又依赖已经揭示的上下文。只匹配最终图像会使监督稀疏且方差较大，逐轮模仿教师则难以真正减少步数，只匹配各位置的边缘分布又会遗漏对象结构和位置相关性。此外，学生推理时会访问教师轨迹之外的错误状态，这种分布偏移可能使误差在少数步骤内迅速累积。

Di[M]O 通过匹配多步教师与一步学生的输出分布，将掩码图像生成模型蒸馏为一步生成器[41]。现有可核验材料没有提供可用于本报告独立复算的同协议质量差值，因此本文不引用其具体实验数字，也不据此推断一步模型在一般情况下必然弱于教师。近期工作进一步尝试学习潜空间受控动力学来跨越多步掩码状态[42]，反映出研究重点正在从“删除步骤”转向“学习跨步状态转移”；[42] 为 2026 年预印本，相关结论限于其公开稿件范围。

一步生成具有最低同步轮数，却可能把全局布局和局部纹理压在同一高难度决策中。少数步生成提供另一种折中：前几步建立可补全结构，后几步在结构条件下恢复属性与细节。这里的关键不只是给总步数设定一个较小值，而是决定学生学习什么。若监督对象是教师终态，学生可能复制高分样本而丢失分布覆盖；若监督对象是教师中间 token 分布，需要保证教师和学生处在可比状态；若监督来自搜索最优样本，还会产生选择偏差，因为高分尾部受到验证器偏好影响。评价少步方法应同时比较固定步数下的质量、固定质量下的真实时延、输出多样性、教师调用成本和训练数据生成成本。

与单条教师轨迹相比，搜索日志还记录了候选之间的相对优劣，以及部分结构经多次补全后的回报。现有测试时扩展主要在每次请求中使用候选信息[5, 36–37, 40]，离散蒸馏则侧重压缩教师生成过程[41–42]。若只保留搜索选出的最终图像，学生容易复制评价器偏好的高分样本；保留候选相对价值和条件补全分布，才可能把预算选择与跨步生成联系起来。

现有代表性工作分别讨论了条件搜索、少步蒸馏和学生状态偏移，但尚未把三者放在同一实验框架中：先用多次补全估计部分结构的价值，再在学生实际访问的状态上构造跨步监督，最后检验搜索收益能否转化为单轨迹的端到端效率。

### 1.7 评价体系、可比性与证据边界

生成优化不能只看一个总分。语义遵循、感知质量、多样性、理解能力和系统开销对应不同评价维度。GenEval 将单对象、双对象、计数、颜色、位置和属性绑定分开统计[1]；DPG-Bench 使用更稠密的提示考察复杂文本遵循，其来源工作为 ELLA[2]。ImageReward、HPSv3、DeQA 和 Unified Reward Model 又从人类偏好或感知质量角度提供补充[34, 43–45]。这些指标各有偏向，因此训练使用的奖励不能同时作为唯一测试指标。

对于统一模型，还需确认生成优化没有损害原有的理解能力。本课题使用 POPE 诊断对象幻觉和 yes-bias，使用 MMBench 覆盖较广的感知与推理能力[46–47]。优化前后保持提示模板与解码设置一致，并结合无图和换图对照，检查模型是否真正利用视觉证据。

对自反馈和多目标奖励，还需要验证评分器本身。应在独立人工标注子集上测量内部问答得分、外部奖励与人类判断的相关性，记录不同错误类型的假阳性和假阴性。训练时使用的问答模板、奖励模型和提示集合不能与最终测试完全重合，否则提升可能来自奖励记忆而非泛化。若多种奖励经加权组合，应报告各分量的尺度、相关性和消融；如果只给加权总分，无法判断模型究竟改善了语义、审美还是迎合某一评分器。人工评价应采用盲评、随机呈现和明确的胜负/平局规则，并报告样本数与置信区间。

效率比较应把验证、解码和数据搬运等开销一并计入。主模型前向次数便于统计，却没有包含问题生成、视觉问答、奖励模型、图像解码、缓存维护和动态批处理。实验将在相同软硬件设置下报告单图延迟、吞吐、峰值显存、主模型与验证器前向次数以及训练 GPU-hours，并分别比较固定预算下的质量和固定质量下的时间。

稳定性与多样性同样属于证据链。奖励优化可能因组内样本分数接近而缺乏有效优势，也可能因奖励差异过大导致策略快速偏离参考模型。应使用多个随机种子报告均值和方差，并监控奖励方差、有效样本量、策略 KL、裁剪率、梯度范数及失败率。对于生成结果，还需检查提示间与同提示多样性，避免以模式收缩换取自动指标。少步蒸馏应区分教师生成数据成本与学生部署成本；推理搜索则应说明候选能否并行及批量变化。没有这些信息，质量—效率结论难以复现。

跨论文数值只适合说明趋势，不宜拼接为统一排行榜。即使使用同名底座，不同 checkpoint、分辨率、CFG、采样器、提示集、检测器版本和随机种子都会改变 GenEval 或 DPG-Bench。所核验的 Lumina-DiMOO 相关稿件存在基础分差异，可能源于模型和协议更新，因而不能用一个论文的基线与另一个论文的方法分数直接相减。可信比较优先采用同一论文同一表中的增量；跨论文比较则重新运行统一评测代码，并公开配置与 checkpoint。

多参考图像创作与编辑可作为文生图之外的外部有效性测试。MICE-Bench 从提示或指令遵循、概念一致性、物理真实性、图像美学与质量、图像结构及非编辑区域一致性等维度评价该任务[48]；本课题仅在资源允许的迁移验证阶段使用，不作为核心结论成立的必要条件。

本文区分正式论文、公开预印本、未发表作者稿和答辩材料，并只在原论文实验协议的范围内引用数据。稿件真实存在不表示其结论已经得到独立验证；个人研究基础也按本人主导、主要参与和团队共同成果分别说明。

### 1.8 国内外研究格局、现有空白与本课题定位

相关研究可以归纳为五条线索：连续扩散、离散视觉表示、统一多模态建模、奖励后训练和推理时扩展。DDPM、潜扩散、VQ-VAE/VQGAN、MaskGIT 和 D3PM 奠定了连续与离散生成基础[6–7, 12–13, 15–16]；Emu3、Show-o 和 Janus-Pro 探索了不同形式的理解—生成统一[18–20]；MMaDA、Muddit、Lumina-DiMOO 和 LaViDa-O 则进一步研究多模态掩码扩散[4, 21–23]。连续扩散中的奖励优化、偏好学习、推理扩展和少步蒸馏为本课题提供了可借鉴的方法[9–11, 29–30, 36–39]。

国内团队参与的 MMaDA、Muddit、Lumina-DiMOO 和 LLaDA-V 覆盖了统一离散底座与视觉指令建模[4, 21–22, 26]，Sketch Then Paint 与 dMLLM-TTS 又把研究推进到层次信用和自验证搜索[3, 5, 40]。相较连续扩散中较成熟的奖励对齐、推理扩展和蒸馏路线[9–11, 29–30, 36–39]，这些工作更关注共享离散主干带来的轨迹定义和内部反馈问题。

在训练阶段，图像级奖励仍难以准确归因到不同 token。随机重掩码、真实轨迹和全掩码代理分别在覆盖范围、轨迹真实性与计算成本上作出取舍[3, 31–32]，但多数方法仍用同一图像级优势更新大量位置。层次权重开始区分结构和细化阶段，却依赖固定比例或生成次序先验。生成秩、预测熵和反事实奖励变化之间是否存在稳定关系，仍需要在统一预算下验证。

在推理阶段，搜索效果受到中间状态评分误差的限制。best-of-N 对完整结果的评价较可靠，但需要重复生成；中间剪枝可以节省计算，却可能把仍可恢复的轨迹提前删除[5, 40]。内部视觉问答适合检查对象、数量和关系，但其在不完整图像上的校准程度，以及评分成本是否抵消主模型节省，尚缺少一致的测量。

自反馈方法还需回答评价器是否真正使用图像证据，以及生成优化会不会损害理解能力。Self-GRPO 等工作说明理解分支可以构造有效的生成奖励[4]，但生成指标提高不能代替理解基准。评价器与生成器共同更新时，奖励标准还可能变化。后续研究需要把内部奖励与人工判断、冻结外部指标、POPE 和 MMBench 等独立测量结合起来[46–47]。

搜索得到的经验如何迁移到少步模型，目前仍缺少系统研究。推理搜索能发现高价值候选，却会在每次请求中重复支付探索与验证成本；一步蒸馏减少在线步骤，也可能损失结构、细节和多样性[5, 40–42]。连续扩散已有轨迹压缩、一致性和分布匹配等方案[9–11]，离散模型还要处理并行 token、可见集合变化和学生状态偏移。

前述问题都与价值估计有关：训练阶段要确定哪些 token 应承担更高信用，搜索阶段要判断哪些轨迹值得继续，反馈阶段要检验评分是否可信，蒸馏阶段要选择可以跨越的中间步骤。本课题把这些问题放在同一质量—成本协议下考察，并通过独立消融判断各部分是否有效。生成顺序和内部反馈都被视为待验证的假设，而不是预先成立的前提。

本课题已有的 HT-GRPO、dMLLM-TTS 与 Self-GRPO 分别为层次信用、轨迹搜索和内部反馈提供阶段性证据[3–5, 40]，粗到细少步生成则构成后续需要完成的环节。相关证据具有不同公开状态：dMLLM-TTS 已正式发表[5]；Lumina-DiMOO 与 Sketch Then Paint 按公开预印本引用[3–4]；dMLLM-TTS v2 为未发表作者稿[40]，答辩材料标注其处于 TPAMI 在审阶段。Lumina-DiMOO 底座属于团队成果，本人主要负责 Self-GRPO，不将整个底座归为个人独立成果。

本课题希望弄清非固定生成顺序中哪些 token 更影响终局结果、内部评价在什么条件下可信，以及搜索经验能否转化为少步模型的固定能力。即使某种顺序先验只在部分提示或底座上成立，界定其适用范围和失败条件仍有研究价值。工程上，将质量、时延、理解能力、多样性和稳定性放在同一协议下报告，可以给出比单一指标更可靠的部署依据。

### 参考文献

[1] Ghosh D, Hajishirzi H, Schmidt L. GenEval: An Object-Focused Framework for Evaluating Text-to-Image Alignment. Advances in Neural Information Processing Systems, 36: 52132-52152, 2023.

[2] Hu X, Wang R, Fang Y, Fu B, Cheng P, Yu G. ELLA: Equip Diffusion Models with LLM for Enhanced Semantic Alignment. arXiv:2403.05135, 2024.

[3] Luo S, Shen J, Xin Y, et al. Sketch Then Paint: Hierarchical Reinforcement Learning for Diffusion Multi-Modal Large Language Models. arXiv:2605.16842, 2026.

[4] Xin Y, Qin Q, Luo S, et al. Lumina-DiMOO: An Omni Diffusion Large Language Model for Multi-Modal Generation and Understanding. arXiv:2510.06308, 2025.

[5] Xin Y, Luo S, Xu T, et al. dMLLM-TTS: Self-Verified and Efficient Test-Time Scaling for Diffusion Multi-Modal Large Language Models. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 35726-35735, 2026.

[6] Ho J, Jain A, Abbeel P. Denoising Diffusion Probabilistic Models. Advances in Neural Information Processing Systems, 33: 6840-6851, 2020.

[7] Rombach R, Blattmann A, Lorenz D, Esser P, Ommer B. High-Resolution Image Synthesis with Latent Diffusion Models. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 2022. DOI: 10.1109/CVPR52688.2022.01042.

[8] Podell D, English Z, Lacey K, Blattmann A, Dockhorn T, Müller J, Penna J, Rombach R. SDXL: Improving Latent Diffusion Models for High-Resolution Image Synthesis. International Conference on Learning Representations, 2024. arXiv:2307.01952.

[9] Salimans T, Ho J. Progressive Distillation for Fast Sampling of Diffusion Models. International Conference on Learning Representations, 2022. arXiv:2202.00512.

[10] Song Y, Dhariwal P, Chen M, Sutskever I. Consistency Models. Proceedings of the 40th International Conference on Machine Learning, PMLR 202: 32211-32252, 2023. arXiv:2303.01469.

[11] Yin T, Gharbi M, Zhang R, Shechtman E, Durand F, Freeman W T, Park T. One-step Diffusion with Distribution Matching Distillation. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 6613-6623, 2024. arXiv:2311.18828.

[12] van den Oord A, Vinyals O, Kavukcuoglu K. Neural Discrete Representation Learning. Advances in Neural Information Processing Systems, 30: 6306-6315, 2017. arXiv:1711.00937.

[13] Esser P, Rombach R, Ommer B. Taming Transformers for High-Resolution Image Synthesis. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 12873-12883, 2021.

[14] Ramesh A, Pavlov M, Goh G, et al. Zero-Shot Text-to-Image Generation. Proceedings of the 38th International Conference on Machine Learning, PMLR 139: 8821-8831, 2021.

[15] Chang H, Zhang H, Jiang L, Liu C, Freeman W T. MaskGIT: Masked Generative Image Transformer. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 2022. DOI: 10.1109/CVPR52688.2022.01103.

[16] Austin J, Johnson D D, Ho J, Tarlow D, van den Berg R. Structured Denoising Diffusion Models in Discrete State-Spaces. Advances in Neural Information Processing Systems, 34: 17981-17993, 2021.

[17] Lou A, Meng C, Ermon S. Discrete Diffusion Modeling by Estimating the Ratios of the Data Distribution. Proceedings of the 41st International Conference on Machine Learning, PMLR 235: 32819-32848, 2024.

[18] Wang X, Zhang X, Luo Z, et al. Emu3: Next-Token Prediction Is All You Need. arXiv:2409.18869, 2024.

[19] Chen X, Wu Z, Liu X, et al. Janus-Pro: Unified Multimodal Understanding and Generation with Data and Model Scaling. arXiv:2501.17811, 2025.

[20] Xie J, Mao W, Bai Z, et al. Show-o: One Single Transformer to Unify Multimodal Understanding and Generation. International Conference on Learning Representations, 2025. arXiv:2408.12528.

[21] Yang L, Tian Y, Li B, Zhang X, Shen K, Tong Y, Wang M. MMaDA: Multimodal Large Diffusion Language Models. Advances in Neural Information Processing Systems, 38, 2025. arXiv:2505.15809.

[22] Shi Q, Bai J, Zhao Z, et al. Muddit: Liberating Generation Beyond Text-to-Image with a Unified Discrete Diffusion Model. International Conference on Learning Representations, 2026. arXiv:2505.23606.

[23] Li S, Gu J, Liu K, et al. LaViDa-O: Elastic Large Masked Diffusion Models for Unified Multimodal Understanding and Generation. arXiv:2509.19244, 2025.

[24] Nie S, Zhu F, You Z, et al. Large Language Diffusion Models. arXiv:2502.09992, 2025.

[25] Gong S, Agarwal S, Zhang Y, et al. Scaling Diffusion Language Models via Adaptation from Autoregressive Models. The Thirteenth International Conference on Learning Representations, 2025. arXiv:2410.17891.

[26] You Z, Nie S, Zhang X, et al. LLaDA-V: Large Language Diffusion Models with Visual Instruction Tuning. arXiv:2505.16933, 2025.

[27] Schulman J, Wolski F, Dhariwal P, Radford A, Klimov O. Proximal Policy Optimization Algorithms. arXiv:1707.06347, 2017.

[28] Shao Z, Wang P, Zhu Q, et al. DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models. arXiv:2402.03300, 2024.

[29] Black K, Janner M, Du Y, Kostrikov I, Levine S. Training Diffusion Models with Reinforcement Learning. International Conference on Learning Representations, 2024. arXiv:2305.13301.

[30] Wallace B, Dang M, Rafailov R, Zhou L, Lou A, Purushwalkam S, Ermon S, Xiong C, Joty S, Naik N. Diffusion Model Alignment Using Direct Preference Optimization. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 2024. arXiv:2311.12908.

[31] Ma T, Zhang M, Wang Y, Ye Q. Consolidating Reinforcement Learning for Multimodal Discrete Diffusion Models. International Conference on Learning Representations, 2026. arXiv:2510.02880.

[32] Wang Y, Yang L, Li B, Tian Y, Shen K, Wang M. Revolutionizing Reinforcement Learning Framework for Diffusion Large Language Models. International Conference on Learning Representations, 2026. arXiv:2509.06949.

[33] Radford A, Kim J W, Hallacy C, et al. Learning Transferable Visual Models from Natural Language Supervision. Proceedings of the 38th International Conference on Machine Learning, PMLR 139: 8748-8763, 2021.

[34] Xu J, Liu X, Wu Y, et al. ImageReward: Learning and Evaluating Human Preferences for Text-to-Image Generation. Advances in Neural Information Processing Systems, 36: 15903-15935, 2023.

[35] Mao W, Yang Z, Shou M Z. UniRL: Self-Improving Unified Multimodal Models via Supervised and Reinforcement Learning. arXiv:2505.23380, 2025.

[36] Ma N, Tong S, Jia H, et al. Scaling Inference Time Compute for Diffusion Models. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 2025.

[37] Singhal R, Horvitz Z, Teehan R, et al. A General Framework for Inference-Time Scaling and Steering of Diffusion Models. arXiv:2501.06848, 2025.

[38] Li S, Kallidromitis K, Gokul A, et al. Reflect-DiT: Inference-Time Scaling for Text-to-Image Diffusion Transformers via In-Context Reflection. Proceedings of the IEEE/CVF International Conference on Computer Vision, 2025. arXiv:2503.12271.

[39] Zhuo L, Zhao L, Paul S, et al. From Reflection to Perfection: Scaling Inference-Time Optimization for Text-to-Image Diffusion Models via Reflection Tuning. Proceedings of the IEEE/CVF International Conference on Computer Vision, 2025. arXiv:2504.16080.

[40] Luo S, Xin Y, Cheng Y, et al. dMLLM-TTS v2: Enhanced Self-Verified and Efficient Test-Time Scaling for Diffusion Multi-Modal Large Language Models. Unpublished manuscript, 2026.

[41] Zhu Y, Wang X, Lathuilière S, Kalogeiton V. Di[M]O: Distilling Masked Diffusion Models into One-Step Generator. Proceedings of the IEEE/CVF International Conference on Computer Vision, 18606-18618, 2025. arXiv:2503.15457.

[42] Zhu K, Zeng Q, Pu Y, et al. Accelerating Masked Image Generation by Learning Latent Controlled Dynamics. arXiv:2602.23996, 2026.

[43] You Z, Cai X, Gu J, Xue T, Dong C. Teaching Large Language Models to Regress Accurate Image Quality Scores Using Score Distribution. Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, 14483-14494, 2025.

[44] Ma Y, Shui Y, Wu X, Sun K, Li H. HPSv3: Towards Wide-Spectrum Human Preference Score. Proceedings of the IEEE/CVF International Conference on Computer Vision, 2025. arXiv:2508.03789.

[45] Wang Y, Zang Y, Li H, Jin C, Wang J. Unified Reward Model for Multimodal Understanding and Generation. arXiv:2503.05236, 2025.

[46] Li Y, Du Y, Zhou K, Wang J, Zhao W X, Wen J R. Evaluating Object Hallucination in Large Vision-Language Models. Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, 292-305, 2023. DOI: 10.18653/v1/2023.emnlp-main.20. arXiv:2305.10355.

[47] Liu Y, Duan H, Zhang Y, Li B, Zhang S, Zhao W, Yuan Y, Wang J, He C, Liu Z, et al. MMBench: Is Your Multi-modal Model an All-around Player? European Conference on Computer Vision, 216-233, 2024. arXiv:2307.06281.

[48] Luo S, Zheng H, Shen J, et al. MICE-Bench: A Challenging and Comprehensive Benchmark for Multi-Reference Image Creation and Editing. Proceedings of the 43rd International Conference on Machine Learning, PMLR 306: 83169-83197, 2026.

[49] Luo S, Yang H, Xin Y, Yi M, Wu G, Zhai G, Liu X. TR-PTS: Task-Relevant Parameter and Token Selection for Efficient Tuning. Proceedings of the IEEE/CVF International Conference on Computer Vision, 4360-4369, 2025.

## 二、课题研究目标、主要研究内容和拟解决的关键问题

### 2.1 总体研究目标

本课题研究有限计算预算下的多模态离散扩散生成。实证工作以文本—图像统一模型为主，先分析非固定去掩码轨迹中的概率近似、价值估计和误差传播，再据此改进训练信用、推理搜索与少步生成。

给定文本条件 c，以 z_t 表示第 t 步完整的视觉 token 状态，以 M_t 表示仍被掩码的位置。状态还记录剩余预算 b_t、采样调度 h_t 和继续策略版本 ν_t。主方案采用单调去掩码：固定日程给出本轮恢复数量，位置集合 S_t 由排序规则确定，随机动作只包括这些位置的 token 类别 y_t(S_t)。完整轨迹记为 𝒯，最终图像由完整 token 状态解码得到。允许重新掩码的方案仅作扩展，并另行定义重掩码集合和状态转移。

[[EQUATION:state_action]]

评价分为三个层次。R* 表示抽象的目标效用；可重复计算且不作为训练奖励的外部指标记为 R_eval，用来近似 R*；训练和搜索使用的模型问答或质量分数记为 R̂_train。人工数据再分成互斥的校准子集和最终审计子集：前者只用于确定相关性与门槛，后者在配置锁定后评估一次。两者都不进入在线条件补全。

给定继续策略 π_cont、剩余预算、采样调度和教师版本，中间状态的预算条件价值定义为：

[[EQUATION:conditional_value]]

该量表示从当前状态继续生成后，最终结果在 R_eval 下的期望得分。候选比较使用同一继续策略、剩余预算、采样调度和教师版本；策略或教师更新后，旧价值不再被当作状态的固有属性，而要重新估计。本文把保留、替换或重新掩码一个决策前后的价值差称为局部反事实敏感度。替换类别从同一步除原类别外的归一化预测分布中采样，后续策略和随机数保持一致。这个量用于比较局部影响，不要求不同 token 的贡献可加，也不作严格因果解释。

第一项理论问题是全掩码代理究竟丢失了什么。主方案的位置集合由确定性排序给出，因此轨迹似然只分解已选位置上的类别条件概率。全掩码对数比与真实状态对数比的差，可以写成两种上下文下类别对数概率之差；在类别 logit 对隐藏状态满足局部 Lipschitz 条件时，拟用相应隐藏表示距离控制这一误差，并用真实轨迹上的 logit 差检验假设。随机位置策略只在扩展实验中使用，届时另行加入位置集合概率。

第二项问题是估值误差何时会改变剪枝结果。若 R̂_train 对 R_eval 的统一误差不超过 ε_R，补全回报为有界或次高斯变量，则固定 P 次补全的误差半径随候选数 J、置信度 1−δ 和 P 变化。两个候选的价值间隔大于两倍总误差时，排序保持不变。在线搜索会反复查看结果并改变 P_t，因此正式方法使用序贯有效的置信序列，而不是普通置信区间；理论与实验共同检查覆盖率和误剪率。

第三项问题是少步误差如何累积。以总变差距离或由 KL 经 Pinsker 不等式得到的距离衡量位置与类别转移差异，在终局效用有界、教师和学生使用同一 tokenizer 的条件下，拟把期望终局效用差分解为教师价值标签误差、各步位置误差、类别误差和学生状态分布偏移的有限步加总。各项都能从教师补全、学生轨迹和消融实验中估计。

总体目标是在固定计算预算下提高生成质量，同时避免理解能力和输出多样性明显下降：

[[EQUATION:constraints]]

预算按用途分别记录。B_train 约束奖励后训练的采样、评价和反向传播；B_serve 约束在线搜索中的主模型、临时补全、问答验证、解码和调度；B_teacher 记录离线教师数据生成；B_deploy 约束少步学生的单轨迹推理。公式中的 C(𝒯) 与 B_comp 是这四组成本和上限的向量简写，按分量比较。各主比较只匹配相应预算，不能用离线教师成本掩盖在线开销。质量与理解非劣界在预实验后确定，并在最终测试前固定。

评价校准为信用分配和轨迹搜索提供共同尺度；动态信用得到的结构先验与搜索记录再共同用于少步学生。四项工作的关系见图 1。

[[FIGURE:unified_value_framework]]

图1 条件价值建模与四项研究的关系

### 2.2 主要研究内容

#### （1）条件价值建模与代理反馈校准

统一模型先把提示拆成对象、数量、颜色和关系问题，再由理解分支检查生成图像。实验用无图、错图和遮挡对照判断评分是否真正依赖视觉内容，并比较内部得分、独立指标与人工判断。随后分析代理误差怎样传到中间状态价值和候选排序。

#### （2）离散生成过程中的分层信用分配

真实轨迹记录生成秩、预测熵和跨步稳定性，少量反事实补全用于估计位置与类别的局部影响。全掩码概率比作为低成本近似，再用真实中间状态测量偏差。实验重点判断生成顺序是否提供了均匀信用和单一置信度之外的信息。

#### （3）基于条件价值的轨迹搜索与预算分配

模型先产生多个结构候选，再对每个候选作少量同预算补全，由固定评价器估计继续生成价值。判断不稳定时增加补全或暂缓剪枝，并保留少量探索候选。搜索规则由序贯有效的置信界控制，所有补全、验证和解码时间都计入在线成本。

#### （4）搜索经验驱动的粗到细少步生成

搜索日志为同一提示保留高、低价值候选及教师后续补全。学生学习恢复位置和相应 token 分布，训练中周期性加入自己的中间状态以减轻分布偏移。部署时只运行一条少步轨迹，并用生成质量和真实延迟判断迁移是否有效。

### 2.3 拟解决的关键问题

第一，怎样定义并估计非固定去掩码轨迹中一个决策的贡献。位置集合与 token 类别共同决定下一状态，但主方案中的位置选择是确定性排序，不能把类别概率比直接解释为完整联合策略概率。本课题将分别建模位置排序和类别预测，并分析全掩码近似相对真实轨迹条件的偏差。

第二，含噪的内部评价能否可靠估计部分状态的最终价值。需要区分当前不完整图像的表面分数与继续补全后的期望结果，并说明代理奖励偏差、补全次数和候选价值间隔怎样共同影响排序。

第三，统一模型的理解分支能否提供依赖图像证据且相对稳定的反馈。评价器可能从提示猜答案，也可能与生成器共享偏差。本课题将通过无图、换图、遮挡和反事实问题检查图像依赖性，并用独立理解基准和人工样本监测漂移。

第四，搜索经验能否迁移为少步生成能力。学生不仅要复制 token 类别，还要学习恢复位置和纠错时机。需要分析教师估计误差、位置与类别蒸馏误差以及学生状态偏移，并与普通蒸馏和直接截短教师进行同预算比较。


## 三、拟采取的研究方法、研究方案及其可行性分析

### 3.1 总体技术路线

先在具有可用权重、代码和评测条件的代表性模型上复现基础推理、奖励后训练、搜索与蒸馏基线。数据划分、分辨率、提示集合、解码和评测实现保持一致。

基线稳定后校准评价器，再分别开展训练信用和推理预算实验。训练端比较均匀、固定层次和动态层次信用；推理端比较单轨迹、best-of-N、线性搜索、随机剪枝和价值引导搜索。最后用搜索日志训练少步学生，并逐项移除评价校准、动态信用、搜索偏好、位置监督和类别监督。主结论先在一个底座完成，资源允许时再验证第二个底座。

### 3.2 条件价值与评价校准

R_eval 由不作为训练奖励的外部自动指标计算。人工校准子集用于设定评价器门槛，互斥的最终人工审计子集只在配置锁定后使用。训练与搜索使用一份独立的评价器参数副本，主方案在一个训练阶段内保持其参数不变；低频同步或指数滑动平均只作对照。评价器只接收图像和问题，不接收原始提示或标准答案。理解分支仅使用有可靠标签的数据更新，避免把错误图像对应的提示答案当作监督。

对固定的继续策略、剩余预算、调度和教师版本，分别定义基于 R_eval 的目标价值 V_eval 与基于 R̂_train 的代理价值 V_train。若代理误差与补全采样误差在同一高概率事件上分别由 ε_R 和 ε_MC 控制，则每个候选的总误差不超过 ε_R+ε_MC；候选价值间隔超过 2(ε_R+ε_MC) 时，排序保持不变。固定样本分析使用候选数校正，序贯搜索使用同一置信水平下的置信序列。

内部语义、感知质量和多样性分开报告。评价器只有同时满足三项条件才进入主训练或搜索：有图得分相对无图、错图有稳定增量；与人工判断的排序相关达到预定下限；POPE 和 MMBench 相对基础模型不超过允许退化范围。具体阈值由预实验和人工最小有意义差异确定，最终测试前锁定。

### 3.3 分层信用分配

设当前状态为 s_t，主方案由固定日程决定恢复数量 m_t，再按位置分数 q_θ(i|s_t,c) 选择前 m_t 个位置。S_t 因而是确定性排序结果，不进入重要性概率比；位置头通过排序损失和反事实敏感度训练。类别分布决定这些位置写入的 token。扩展实验若采用随机无放回选择，则单独定义 p_θ(S_t|s_t,c)，并与类别概率组成联合轨迹似然。

位置分数使用生成秩、预测熵、跨步稳定性、空间信息和局部反事实敏感度。类别替换从同一步的备选分布采样；位置干预则交换边界附近候选。两种比较都固定继续策略并使用配对随机数。所得敏感度只用于排序和加权，不要求所有 token 权重之和还原终局奖励。图像级优势与 token 权重分别归一化，使不同方法具有相近的平均梯度尺度。

全掩码代理在仅文本可见时计算已选位置上新旧策略的类别概率比，真实轨迹类别比则以当前部分可见状态为条件。二者的差异只用于分析上下文近似，不包含确定性位置选择的概率。实验在抽样状态上测量对数比偏差、相关性、裁剪率和策略 KL。

### 3.4 条件价值引导的轨迹搜索

搜索先产生 N 个结构候选，并把候选对齐到相同阶段、剩余预算、调度和教师版本。每个候选先独立补全少量样本，得到可解码图像并由固定评价器打分。均值估计条件价值，序贯有效置信序列衡量不确定性。区间仍重叠的候选继续补全，明显落后的候选停止，同时保留少量随机探索分支。

理论分析在回报有界或条件次高斯、代理误差在校准状态集上一致受控的条件下，给出补全次数、候选数、置信水平与误剪概率的关系。自适应 P_t 使用 anytime-valid 置信序列并对多候选比较分配错误概率。实际回报的尾部、区间覆盖率和误剪率用于检验假设；早期状态不满足条件时推迟剪枝。

复杂度按实际候选宽度、补全次数和分支长度计算。端到端成本还包括提示解析、问题生成、视觉问答、图像解码、数据搬运和动态批处理。实验同时报告主模型前向次数、GPU 运行时间、P50/P95 延迟、吞吐和峰值显存。

比较对象包括单轨迹、best-of-N、线性完整搜索、同预算随机剪枝、同评分器均匀预算和冻结外部评价器。独立测试指标只用于最终分析，不参与候选选择。对抽取的训练候选，无论搜索时是否淘汰，都继续生成至终点，以获得可比较的教师回报。

### 3.5 粗到细少步生成

训练数据包含同一提示下多个结构候选、在线估计价值、搜索保留结果及冻结教师的后续补全。每个结构由教师独立补全 M 次，平均代理效用作为训练价值，并保留低价值候选帮助学生学习决策边界。M 根据预实验中的方差与成本关系确定。

研究二得到的位置头用于初始化学生的位置排序，提示相关的结构概率作为辅助排序目标和候选样本权重。去掉该迁移、只保留搜索日志的学生作为消融，以判断动态信用是否确实帮助少步学习。

学生从搜索状态和自己生成的状态出发，同时学习位置与类别。位置目标由教师在跨步窗口内恢复某位置的频率构成；类别目标汇总教师真正恢复该位置前的软分布。主方案保持已写入位置不变。重掩码只作扩展对照，并显式记录重掩码集合 D_t 与相应状态转移，不纳入主方案的轨迹理论。总目标由候选排序、位置蒸馏、类别蒸馏和参考策略正则组成。

理论分析用总变差距离衡量位置和类别转移差异，并用 KL 与 Pinsker 不等式给出可训练的替代量。在终局效用有界、教师与学生共享 tokenizer、总步数有限的条件下，推导期望终局效用差关于价值标签误差、逐步位置误差、类别误差和状态分布偏移的递推上界。逐项消融和学生在线状态刷新用于判断哪一项主导实际误差。

推理时固定总步数，比较均匀、前多后少、前少后多和按不确定性调整的预算日程。基线包括完整教师、教师直接截短、一步 Di[M]O 类方法、相同步数普通蒸馏、纯结构强化和先蒸馏后强化。少步方法只有在质量满足预设范围且单轨迹实际时间下降时，才被视为实现了部署加速。

### 3.6 实验设计与评价

除当前研究的变量外，各方法使用相同底座、tokenizer、分辨率、提示集合、引导系数和评测代码。训练实验匹配 B_train，在线搜索匹配 B_serve，教师数据另报 B_teacher，少步部署在 B_deploy 下比较。采样或位置日程本身作为研究变量时，不要求强行相同。

| 研究内容 | 主要比较 | 通过判据 |
|---|---|---|
| 价值与评价 | 内部评价、外部评价、人工校准子集 | 有图相对无图/错图的得分差达到阈值；人工排序相关达到下限；留出状态上 V_train 与 V_eval 的绝对差或价值区间覆盖误差不超过上限；POPE、MMBench 分别满足非劣界 |
| 分层信用 | 均匀、随机、固定层次、动态层次 | 匹配 B_train 后，GenEval Overall 配对差值区间下界高于 0；策略 KL 和多样性满足预设范围 |
| 轨迹搜索 | 单轨迹、best-of-N、线性与随机搜索 | 相对同 B_serve 的线性完整搜索，质量差区间下界高于 −δ_G，且端到端时间或 GPU-seconds 差区间上界低于 0 |
| 少步生成 | 普通蒸馏、教师截短、一步基线 | 优于相同步数普通蒸馏，教师质量差高于 −δ_G，且单轨迹实际时间下降；三项同时满足 |

生成质量主要使用 GenEval 和 DPG-Bench，感知质量与偏好指标作为补充。理解能力使用 POPE 和 MMBench。统一成本记录包括主模型与评价器前向、GPU-seconds、延迟、吞吐、显存和训练 GPU-hours，后文不再另设口径。人工评价采用盲评、随机左右顺序并允许平局。

关键机制至少进行三次独立训练；同一模型增加生成随机种子只用于估计采样方差。方法尽量共享提示和采样随机数，主要差值使用配对 bootstrap 或适当的混合效应模型估计区间。非劣界和主要判据在查看最终测试结果前确定。失败判据集中记录在实验方案中，不在各章节重复。

### 3.7 可行性与风险

前期 HT-GRPO 训练使用过 8 张 A100-80G，Lumina-DiMOO 的 Self-GRPO 使用过 8 张 H200，说明团队已有同类训练经验。后续可用算力仍需在正式实验前与导师确认，并据此确定模型规模和重复次数。

算力最先限制的是多次条件补全和学生状态刷新。资源不足时，优先保留条件价值误差分析、动态信用和一个底座上的搜索—蒸馏主实验；第二底座和多参考任务后移。GPU-hours、搜索宽度和样本量在目标机器完成预实验后确定，并记录实际吞吐、峰值显存和失败率。

全掩码代理偏差过大时，改用部分上下文或真实状态校准；内部评价过度依赖文本先验时，不进入主训练与搜索，并改用冻结外部评价作对照；序贯假设不成立时，把置信序列改为经验搜索启发式并取消概率保证；少步学生质量下降时，改用分阶段训练并报告质量—步数曲线。生成顺序、自评分或搜索迁移若没有稳定增量，相应结论只保留为适用边界。


## 四、课题的创新点

### 4.1 条件价值与含噪生成决策的误差分析

新增的理论部分区分目标评价、训练代理和中间状态价值，并围绕三项可检验命题展开：全掩码类别对数比的上下文偏差，序贯补全中的估值误差与误剪概率，以及少步学生在位置、类别和状态分布上的有限步误差传播。每项命题都给出假设、可观测量和实验检验方式。

### 4.2 面向生成层次的动态信用分配

HT-GRPO 已用固定阶段权重处理生成信用[3]。本课题新增提示相关的位置排序和局部反事实敏感度，用真实中间状态测量全掩码代理偏差，并分别训练位置排序与类别预测。结果通过同预算对照判断，不再把固定生成次序直接当作结构标签。

### 4.3 基于自验证价值的轨迹预算分配

dMLLM-TTS 已证明自验证可以用于轨迹筛选[5]。本课题新增多次条件补全、序贯有效的不确定性控制和图像证据检验，使搜索依据从当前图像分数转为继续生成价值。效率按完整在线时间核算。

### 4.4 搜索经验驱动的粗到细少步生成

现有离散少步方法主要压缩教师生成过程[41–42]。本课题把搜索中的高、低价值候选一并用于训练，在学生实际访问的状态上同时蒸馏恢复位置和 token 类别，并用单轨迹部署检验搜索经验是否转化为固定预算能力。


## 五、计划进度、预期成果

### 5.1 计划进度

2026 年 10 月—12 月：完成基础结果复现、研究边界和实验口径确认。重点核对前期模型、评价指标与搜索实现，并在目标软硬件上测量吞吐、显存和生成成本，据此确定后续实验规模。

2027 年 1 月—4 月：完成理论分析和小规模机制实验。分析全掩码代理偏差、条件价值估计误差与误剪概率，以及跨步蒸馏误差来源；同步完成无图、换图和反事实问题等评价器检查。

2027 年 5 月—8 月：在一个代表性多模态离散扩散模型上完成条件价值与评价校准、分层信用、轨迹搜索和少步生成的主实验。统一报告生成质量、理解能力和实际运行成本。

2027 年 9 月—12 月：开展交叉消融和迁移验证。逐项移除评价校准、动态信用、搜索偏好、位置监督和类别监督。资源允许时增加第二底座、另一分辨率或多参考图像任务；若迁移效果不稳定，则把结论限定在主底座。

2028 年 1 月—3 月：锁定实验配置并运行最终测试，完成统计分析、人工盲评和失败案例整理。统一前期论文与新增工作的术语、模型版本和个人贡献边界，形成学位论文初稿和预答辩材料。

2028 年 4 月—6 月：根据预答辩和送审意见补充必要实验，完成论文定稿、答辩和代码归档，并再次核对论文发表状态、个人贡献和参考文献。

### 5.2 预期成果与验收方式

预期成果包括三项理论分析、四个对应算法模块和一套统一实验报告。理论结果分别处理全掩码代理偏差、序贯价值估计与误剪概率、少步转移误差传播；算法模块包括评价校准、动态信用、价值搜索和位置—类别联合蒸馏。验收材料保留模型版本、配置、评测脚本、四类预算、质量—效率曲线和失败案例。

最终成果包括博士学位论文《面向多模态离散扩散模型的生成优化理论与方法》、相关论文或投稿材料、统一评测脚本、实验日志、质量—计算曲线及可公开范围内的代码和模型。主实验规模以实际算力为准；若只能完成较小模型实验，论文将明确把结论限定在机制验证范围内。


## 六、与本课题有关的工作积累、已有的研究工作成绩

前期工作已经留下可复用的统一离散扩散模型、奖励训练代码、轨迹搜索实现和生成评测流程，能够直接支持本课题的评价校准、信用分配和少步蒸馏实验。

在 Lumina-DiMOO 中，本人作为共同第一作者，主要负责 Self-GRPO 自反馈优化。该方法用生成图像上的结构化问答构造语义奖励，是研究一的直接基础。本课题将在此基础上增加阶段内固定的评价副本、低频同步对照、有图与无图/错图条件下的得分比较，以及独立理解能力约束。统一底座、tokenizer 和通用训练设施属于团队共同成果。答辩材料标注该论文处于 IJCV Minor Revision 阶段。

在推理时扩展方向，本人作为共同第一作者参与 dMLLM-TTS（CVPR 2026），并作为第一作者开展 dMLLM-TTS v2。前者实现自验证和分层轨迹搜索，后者扩展多调度探索与多维评价。现有实现可直接用于研究三的候选生成和搜索基线；新增工作是中间状态价值、序贯不确定性控制和完整在线成本。答辩材料标注 v2 处于 TPAMI 在审阶段。

在离散生成强化学习方向，本人作为第一作者完成 Sketch Then Paint，提出 HT-GRPO 的三阶段信用分配和全掩码概率比估计。它为研究二提供训练代码和固定层次基线；本课题新增动态位置排序、局部反事实敏感度及代理偏差分析。答辩材料标注该工作已被 NeurIPS 2026 录用，所附稿件为较早的匿名版本。

本人还以第一署名共同第一作者完成 MICE-Bench（ICML 2026）和 TR-PTS（ICCV 2025）[48–49]。前者提供复杂生成任务与评价经验，后者涉及参数和 token 层面的计算分配。答辩材料另标注 Enhancing Test Time Adaptation with Few-Shot Guidance 处于 Pattern Recognition Major Revision 阶段。

上述工作覆盖了本课题需要的模型训练、奖励构造、搜索实现和评测环节。现有代码可用于复现基线，新增的条件价值、序贯搜索和少步学生将在同一工程框架中实现。

### 6.1 个人贡献边界与核查计划

下表对照前期成果与本课题新增工作。个人分工只按现有材料填写，尚缺记录的部分在答辩前由本人、导师和合作者核实。

表1 前期工作个人贡献边界与核查计划

| 前期工作 | 已有基础 | 本课题新增 | 个人贡献核查 |
|---|---|---|---|
| Lumina-DiMOO / Self-GRPO | 统一底座与结构化问答奖励 | 评价副本、图像证据校准、理解能力约束 | 本人共同第一作者并主要负责 Self-GRPO；底座与通用设施为团队成果 |
| dMLLM-TTS / v2 | 自验证、分层搜索、多调度探索 | 条件价值、序贯不确定性控制、完整在线成本 | dMLLM-TTS 共同第一作者，v2 第一作者；细分工以提交和实验记录核实 |
| Sketch Then Paint | 固定层次信用与全掩码代理 | 动态位置排序、局部反事实敏感度、代理偏差分析 | 本人第一作者；代码、实验与写作分工答辩前核实 |
| 粗到细蒸馏 | 现有离散蒸馏与本课题搜索日志 | 位置—类别联合蒸馏、学生状态刷新、误差传播分析 | 拟由本人主导；复用团队底座与训练设施 |

学位论文的个人贡献以代码提交、实验记录、配置和论文贡献说明为依据；团队共同完成的底座、数据工程和通用设施单独列明。表中待核部分将在正式答辩前补齐。
