
## Dreamer 
Dreamer 生成潜在状态的过程依托**世界模型的三个核心组件**，通过**表示模型、转移模型和奖励模型**的协同实现，具体步骤如下：


### 1. **表示模型：观测→连续潜在状态**
   - **输入**：原始观测（如图像）\( o_t \) 和历史动作 \( a_{t-1} \)、历史潜在状态 \( s_{t-1} \)。
   - **编码**：使用卷积神经网络（CNN）提取图像特征，结合递归状态空间模型（RSSM）将观测和动作编码为**连续向量 \( s_t \)**，建模为高斯分布 \( p_\theta(s_t | s_{t-1}, a_{t-1}, o_t) \)。
   - **训练目标**：最大化变分下界（ELBO），通过**图像重构损失**（像素级重建）和**KL散度正则化**，强制潜在状态捕捉观测中与动态相关的信息。
   - **示例**：在DeepMind Control Suite的视觉任务中，64×64的图像经CNN压缩后，与动作拼接输入RSSM，生成30维的潜在状态，保留运动轨迹的时序信息。


### 2. **转移模型：潜在状态的时序预测**
   - **动态建模**：RSSM作为转移模型 \( q_\theta(s_t | s_{t-1}, a_{t-1}) \)，基于前序状态和动作，预测下一潜在状态的高斯分布。
   - **想象轨迹**：在训练和推理中，通过采样或均值预测生成未来潜在状态序列，支持多步想象（如 horizon \( H=15 \)），用于价值估计和策略优化。
   - **关键创新**：递归结构捕捉长期依赖，使潜在状态具备马尔可夫性，避免历史观测的重复编码，降低计算成本。


### 3. **奖励模型：潜在状态→奖励预测**
   - **映射关系**：潜在状态 \( s_t \) 输入全连接网络，输出奖励预测 \( q_\theta(r_t | s_t) \)，直接关联潜在空间与任务目标。
   - **训练协同**：与表示模型、转移模型联合优化，通过重构奖励信号（\( \mathcal{J}_R^t \)）引导潜在状态编码与奖励相关的特征。


### 4. **训练机制：重构驱动的表征学习**
   - **损失函数**：总损失包含图像重构（\( \mathcal{J}_O^t \)）、奖励重构（\( \mathcal{J}_R^t \)）和KL散度（\( \mathcal{J}_D^t \)），强制潜在状态在压缩信息的同时保留预测未来的能力。
   - **对比实验**：纯奖励预测（无图像重构）表现最差，而结合图像重构（如RSSM）显著提升潜在状态的表征质量，验证视觉信息对动态建模的重要性。


### 5. **潜在状态的应用：想象与决策**
   - **策略优化**：通过潜在状态想象轨迹（\( s_{\tau}, a_{\tau} \)），计算多步价值估计 \( V_\lambda \)，反向传播梯度更新策略网络，实现“ latent imagination ”。
   - **长期依赖**：RSSM的递归结构使潜在状态隐含历史信息，支持长时序预测（如45步的视频重构，图3-57），解决视觉控制中的延迟奖励问题。


### 总结：Dreamer的潜在状态生成逻辑

观测和动作经CNN+RSSM编码为连续潜在状态，通过重构损失和动态预测训练，最终用于想象轨迹以优化策略。其核心是**通过视觉重构强制潜在状态捕获环境动态**，而非仅依赖奖励信号，从而在高维视觉任务中实现高效的模型-based RL。




# DC-MPC
DC-MPC（Discrete Codebook Model Predictive Control）生成潜在状态的过程基于**离散码本世界模型（DCWM）**，通过**编码器→量化→动态建模**三步实现，具体操作如下：


### **1. 编码器：观测值→连续潜在向量**
- **输入**：原始观测值 \( o_t \in \mathbb{R}^{obs\_dim} \)（如DMControl中Humanoid的67维状态）。
- **网络结构**：多层感知机（MLP），含归一化层（LayerNorm）和Mish激活函数，输出连续潜在向量 \( x = e_\theta(o_t) \in \mathbb{R}^{b \times d} \)（默认 \( b=2 \) 通道，\( d=512 \) 维度）。
- **作用**：提取观测中的关键特征，为离散化做准备。


### **2. 量化：连续向量→离散码本编码**
- **FSQ量化**：使用**有限标量量化（FSQ）**将每个通道独立量化为离散码：
  - **量化级别**：每个通道预设量化等级 \( L_i \)（默认 \( L=[5,3] \)，即第1通道5级，第2通道3级）。
  - **离散化公式**：  
    \[
    c_i = \text{round\_ste}\left( \left\lfloor \frac{L_i}{2} \right\rfloor \cdot \tanh(x_i) \right)
    \]  
    其中 \( \text{round\_ste} \) 为直通过程（STE），允许梯度反向传播，\( \tanh \) 归一化到 [-1, 1]。
  - **码本结构**：生成 \( |C| = \prod L_i = 15 \) 个离散码，每个码为 \( b \) 维整数向量（如 [0, -1], [2, 1] 等），保留多维度序数关系（如 \( c1_x > c2_x \) 但 \( c1_y < c2_y \)）。
- **优势**：相比DreamerV3的one-hot编码，码本编码维度低（15 vs. 高维稀疏），且支持连续控制中的序数关系建模。


### **3. 动态建模：离散状态转移**
- **输入**：当前离散码 \( c_t \) 和动作 \( a_t \)。
- **动力学模型**：MLP分类器 \( d_\phi(c_t, a_t) \) 输出下一状态的logits，通过Softmax得到类别分布 \( p(c_{t+1} | c_t, a_t) \)。
- **训练目标**：交叉熵损失（CE Loss），最小化预测分布与真实码 \( c_{t+1}^* = f(e_\theta(o_{t+1})) \) 的差异。
- **多步预测**：使用**ST Gumbel-softmax采样**，在训练时通过噪声注入（Gumbel分布）保持随机性，同时允许梯度回传至编码器，优化长期依赖。


### **4. 规划阶段：确定性期望状态**
- **推理优化**：规划时不采样离散码，而是计算期望状态 \( \hat{c}_{t+1} = \sum p(c^{(i)}|c_t,a_t) \cdot c^{(i)} \)，通过加权平均实现离散码的“插值”，避免随机采样的方差问题。
- **闭环控制**：结合MPPI算法，利用期望状态预测未来轨迹，优化动作序列（如Humanoid行走时的关节角度协调）。


### **关键创新点**
- **离散vs.连续**：通过FSQ量化将连续状态空间划分为离散码，利用分类损失（CE）替代回归（MSE），减少多步预测的误差累积（图3对比实验）。
- **码本设计**：固定码本+序数关系保留，比VQ（向量量化）更稳定，无需字典学习（图10 ablation）。
- **训练策略**：联合训练编码器、动力学、奖励模型（式1-64），通过N-step TD和REDQ评论家提升值函数估计精度。


### **示例：Humanoid Walk任务**
1. **编码**：将67维关节角度/速度输入编码器，输出2×512的连续向量。
2. **量化**：第一通道（5级）量化关节角度范围，第二通道（3级）量化角速度，生成如 [3, -1] 的离散码，表示“中等前倾角度+负向角速度”。
3. **动态预测**：根据当前码和动作（如“右膝弯曲”），动力学模型预测下一码为 [2, 0]，对应“小角度前倾+零角速度”，优化行走稳定性。


### **总结**
DC-MPC通过**编码器提取特征→FSQ量化离散码→分类器建模动态**，生成兼具离散效率和连续序数关系的潜在状态，在高维控制任务（如Dog Run、Humanoid）中实现样本高效学习（图13）。其核心优势在于离散码本的结构化设计，避免了连续空间的模糊性和one-hot的高维稀疏问题，为模型预测控制提供了更可靠的状态表征。