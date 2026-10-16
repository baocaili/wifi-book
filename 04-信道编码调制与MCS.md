# 第 4 章　信道编码、调制与 MCS ★

## 学习目标

1. 掌握 二进制卷积码（Binary Convolutional Coding, BCC）与 低密度奇偶校验码（Low-Density Parity-Check, LDPC）的结构、译码原理与性能差异。
2. 能由 调制编码方案（Modulation and Coding Scheme, MCS）索引、带宽、流数、保护间隔（Guard Interval, GI）精确计算 物理层（Physical Layer, PHY）速率。
3. 理解 误差矢量幅度（Error Vector Magnitude, EVM）、相位噪声、PA 非线性对高阶调制的限制。
4. 能从灵敏度出发计算 MCS 的"可用距离"，绘制 Rate-vs-Range 曲线。

---

## 4.1 信道编码

### 4.1.1 BCC（二进制卷积码）

802.11 使用 $K = 7$（约束长度）、码率 1/2 的卷积码，生成多项式 $(133_8, 171_8)$，通过**打孔（puncturing）**得到 2/3、3/4、5/6 码率。

**特点**：
- 编译码简单，Viterbi 译码复杂度 $O(2^{K-1}) = O(64)$ 状态
- 11n/ac 时代主流，11ax 起仅在低速率与信号字段（Signal field, SIG）字段使用（SIG 字段必须用 BCC，以保证可靠性）
- 缺点：长码块性能不如 LDPC，且不支持很高码率

**打孔表**（11a/g/n，2/3 示例）：每 4 个输入比特产生 6 个输出比特，删去 2 个。

### 4.1.2 LDPC（低密度奇偶校验码）

11ac/ax/be 引入，成为**数据字段的主力编码**。

**结构**：802.11 LDPC 使用 **QC-LDPC（准循环）** 结构，校验矩阵由 $Z\times Z$ 的循环移位矩阵构成，便于硬件并行译码。

支持码块长度：648、1296、1944 bit，码率 1/2、2/3、3/4、5/6。

**译码算法**：置信传播（BP / Sum-Product）或简化的最小和（Min-Sum）算法。

**BP 算法推导**：

设校验矩阵 $\mathbf{H}$，变量节点 $v$ 与校验节点 $c$。消息传递在对数似然比（LLR）域进行：

$$
L(q_{v\to c}) = L(y_v) + \sum_{c' \in \mathcal{N}(v)\setminus c} L(r_{c'\to v})
$$

$$
L(r_{c\to v}) = 2\tanh^{-1}\left(\prod_{v' \in \mathcal{N}(c)\setminus v} \tanh\frac{L(q_{v'\to c})}{2}\right)
$$

**Min-Sum 近似**：用 $\tanh^{-1}(\prod \tanh(\cdot))$ ≈ $\min|\cdot| \times \prod \text{sign}(\cdot)$ 替换，避免 $\tanh$ 的硬件开销，性能损失约 0.2–0.5 dB。

**伪代码 4.1　Min-Sum LDPC 译码**

```python
def ldpc_min_sum(Lch, H, max_iter=20, alpha=0.8):
    """
    输入: Lch - 信道 LLR (n,), H - 校验矩阵 (m x n, 0/1)
          alpha - 归一化因子(抵消 Min-Sum 过高估计)
    输出: 译码后 LLR
    """
    m, n = H.shape
    Lq = np.zeros((m, n))       # 变量->校验 消息
    Lr = np.zeros((m, n))       # 校验->变量 消息
    # 初始化
    for (i, j) in zip(*np.nonzero(H)):
        Lq[i, j] = Lch[j]
    for it in range(max_iter):
        # 校验节点更新 (Min-Sum)
        for i in range(m):
            cols = np.nonzero(H[i])[0]
            for j in cols:
                others = cols[cols != j]
                sgn = np.prod(np.sign(Lq[i, others]))
                mag = alpha * np.min(np.abs(Lq[i, others]))
                Lr[i, j] = sgn * mag
        # 变量节点更新
        for j in range(n):
            rows = np.nonzero(H[:, j])[0]
            total = Lch[j] + np.sum(Lr[rows, j])
            for i in rows:
                Lq[i, j] = total - Lr[i, j]
        # 硬判决与校验
        Lpost = Lch + np.sum(Lr, axis=0)
        bits = (Lpost < 0).astype(int)
        if np.all(H @ bits % 2 == 0):
            return Lpost, it + 1            # 校验通过, 提前退出
    return Lpost, max_iter
```

**复杂度**：每轮迭代 $O(d_c \cdot n)$（$d_c$ 为校验节点度，802.11 LDPC 中 $d_c \approx 12$），20 轮迭代约 $240n$ 次运算。对 1944 bit 码块约 47 万次运算——这是高阶调制高速率下芯片面积与功耗的主要来源之一。

### 4.1.3 BCC vs LDPC 性能

| 指标 | BCC | LDPC |
|---|---|---|
| 编码增益（同码率，1944 bit） | 基准 | **+1.5 ~ 2.5 dB** @ 包误码率（Packet Error Rate, PER） $10^{-2}$ |
| 译码复杂度 | 低（Viterbi） | 中高（迭代） |
| 错误平层（error floor） | 无 | 存在（需精心设计 H） |
| 短包性能 | 好 | 略差 |
| Wi-Fi 用途 | SIG 字段、低速率 | 数据字段主力 |

**为什么 SIG 字段仍用 BCC？** SIG 字段短（数十 bit）、必须极高可靠（错了整帧丢失），且 LDPC 在短块下优势不明显、译码延迟更长。因此协议规定 SIG 用 BCC 1/2 + 重复编码。

---

## 4.2 调制

### 4.2.1 星座与比特映射

| 调制 | $N_{bpscs}$ | 每符号能量归一化 | 最小距离 |
|---|---|---|---|
| BPSK | 1 | 1 | $2\sqrt{E_s}$ |
| 四相移键控（Quadrature Phase Shift Keying, QPSK） | 2 | 1 | $\sqrt{2E_s}$ |
| 16-QAM | 4 | $\sqrt{10}$ | $\sqrt{E_s/5}\times 2$ |
| 64-QAM | 6 | $\sqrt{42}$ | $\sqrt{E_s/21}\times 2$ |
| 256-QAM | 8 | $\sqrt{170}$ | $\sqrt{E_s/85}\times 2$ |
| 1024-QAM | 10 | $\sqrt{682}$ | $\sqrt{E_s/341}\times 2$ |
| 4096-QAM | 12 | $\sqrt{2730}$ | $\sqrt{E_s/1365}\times 2$ |

**最小距离递减规律**：每提升一阶（+2 bit），最小距离缩小约 $\sqrt{4} = 2$ 倍 → 需要 **6 dB** 额外 信噪比（Signal-to-Noise Ratio, SNR）维持同样的误码率。这就是"4096-QAM 比 1024-QAM 需要多 6 dB"的来源。

### 4.2.2 误码率近似

MQAM 在高 SNR 下的符号错误率（SER）近似：

$$
P_s \approx 4\left(1 - \frac{1}{\sqrt{M}}\right) Q\left(\sqrt{\frac{3\log_2 M}{M-1}\gamma_s}\right)
$$

其中 $\gamma_s$ 为符号 SNR。格雷映射下 $P_b \approx P_s / \log_2 M$。

**算例 4.1**：求 4096-QAM（$M = 4096$）在 $\gamma_s = 35$ dB 下的 SER。

$\gamma_s = 10^{3.5} = 3162$。

$$
\frac{3\log_2 M}{M-1} = \frac{3\times12}{4095} = 0.00879
$$

$$
\sqrt{0.00879 \times 3162} = \sqrt{27.79} = 5.27
$$

$$
P_s \approx 4(1 - 1/64) Q(5.27) = 3.94 \times 6.8\times10^{-8} = 2.7\times10^{-7}
$$

对应 $P_b \approx 2.2\times10^{-8}$。看似极小，但注意这是**理想 加性白高斯噪声（Additive White Gaussian Noise, AWGN）、无编码、无 EVM 损伤**的结果。实际中 EVM、相位噪声、量化噪声会把有效 SNR 拉低 10 dB 以上。

### 4.2.3 EVM 与 SNR 的等效关系

EVM 造成的性能损失可等效为 SNR 降低：

$$
\gamma_{\rm eff} = \frac{\gamma}{1 + \gamma \cdot \text{EVM}^2}
$$

其中 EVM 为**线性值**（非 dB）。

**关键结论**：当 $\gamma \cdot \text{EVM}^2 \gg 1$ 时，

$$
\gamma_{\rm eff} \approx \frac{1}{\text{EVM}^2}
$$

即 **EVM 设定了 SNR 的天花板**，无论怎么提高发射功率都无法超越。

**算例 4.2**：EVM = −34 dB（4096-QAM 要求），线性值 $= 10^{-3.4} = 3.98\times10^{-4}$。

$$
\gamma_{\rm eff, max} = 1/(3.98\times10^{-4})^2 = 1/1.585\times10^{-7} = 6.31\times10^{6} = 68 \text{ dB}
$$

看似很高，但这是**总的** EVM。若发射机 EVM 已占 −34 dB，接收机还有相位噪声、模数转换器（Analog-to-Digital Converter, ADC）量化、信道估计误差……叠加后总 EVM 通常只能到 −32 dB，对应有效 SNR 上限约 60 dB。而实际环境中 3 米外的路径损耗已使 SNR 低于 40 dB，因此 4096-QAM 在更远处完全不可用。

---

## 4.3 MCS 表

### 4.3.1 11be（EHT）MCS 表

| MCS | 调制 | 码率 | $N_{bpscs}$ | 20 MHz 1SS (GI=0.8μs) 速率 | 典型所需 SNR (dB) |
|---|---|---|---|---|---|
| 0 | BPSK | 1/2 | 1 | 8.6 Mbps | ~3 |
| 1 | QPSK | 1/2 | 2 | 17.2 | ~6 |
| 2 | QPSK | 3/4 | 2 | 25.8 | ~9 |
| 3 | 16-QAM | 1/2 | 4 | 34.4 | ~12 |
| 4 | 16-QAM | 3/4 | 4 | 51.6 | ~15 |
| 5 | 64-QAM | 2/3 | 6 | 68.8 | ~19 |
| 6 | 64-QAM | 3/4 | 6 | 77.4 | ~20 |
| 7 | 64-QAM | 5/6 | 6 | 86.0 | ~22 |
| 8 | 256-QAM | 3/4 | 8 | 114.7 | ~26 |
| 9 | 256-QAM | 5/6 | 8 | 127.5 | ~28 |
| 10 | 1024-QAM | 3/4 | 10 | 143.3 | ~32 |
| 11 | 1024-QAM | 5/6 | 10 | 159.2 | ~34 |
| 12 | 4096-QAM | 3/4 | 12 | 172.1 | ~38 |
| 13 | 4096-QAM | 5/6 | 12 | 191.2 | ~41 |

（SNR 门限为典型实现值，随 EVM、实现损耗、信道模型变化，仅供量级参考。）

### 4.3.2 速率公式

$$
\boxed{R = \frac{N_{sd} \cdot N_{bpscs} \cdot R_c \cdot N_{ss}}{T_{sym}}}
$$

其中 $T_{sym} = T_{DFT} + T_{GI}$。

**推导说明**：
- 每 正交频分复用（Orthogonal Frequency Division Multiplexing, OFDM）符号承载 $N_{sd}$ 个子载波 × $N_{bpscs}$ 个**编码后**比特？
  **注意**：$N_{bpscs}$ 定义是"每子载波每空间流的**编码比特**数"，即已含码率。查表时 MCS 的 $N_{bpscs}$ 列给出的是调制阶数对应的比特数（如 MCS 9 为 8），速率计算时再乘码率 $R_c$。
- 每符号信息比特 $= N_{sd} \times N_{bpscs} \times R_c \times N_{ss}$
- 每秒符号数 $= 1/T_{sym}$

**算例 4.3**：80 MHz，2 SS，MCS 11，GI = 0.8 μs。

$N_{sd} = 980$，$N_{bpscs} = 10$，$R_c = 5/6$，$N_{ss} = 2$，$T_{sym} = 13.6$ μs。

$$
R = \frac{980 \times 10 \times (5/6) \times 2}{13.6\times10^{-6}} = \frac{980 \times 16.667}{13.6\times10^{-6}} = \frac{16333}{13.6\times10^{-6}} = 1.201 \text{ Gbps}
$$

**算例 4.4（Medium Access Control, MAC 层效率折算）**：上例 PHY 速率 1.201 Gbps，求 传输控制协议（Transmission Control Protocol, TCP）有效吞吐。

扣除：
- 前导 + SIG：约 40 μs（每 物理层汇聚协议（Physical Layer Convergence Protocol, PLCP）协议数据单元（PLCP Protocol Data Unit, PPDU）固定）
- MAC 头 + 加密头：约 36 字节
- MPDU 聚合（Aggregated MPDU, A-MPDU）分隔符 + 填充：每 MAC 协议数据单元（MAC Protocol Data Unit, MPDU） 4 字节
- 确认帧（Acknowledgement, ACK） / BlockAck：约 32 μs + 短帧间间隔（Short Inter-Frame Space, SIFS） 16 μs
- 分布式协调功能（Distributed Coordination Function, DCF）帧间间隔（DCF Inter-Frame Space, DIFS） + 平均退避：$34 + 67.5 = 101.5$ μs（CWmin=15，平均退避 7.5 slot × 9 μs）

设 A-MPDU 包含 64 个 1500 字节帧，聚合长度 $64 \times 1544 = 98{,}816$ 字节。

数据部分时长 $= 98816 \times 8 / 1.201\times10^9 = 658$ μs

总时长 $= 40 + 658 + 16 + 32 + 16 + 101.5 = 863.5$ μs

有效吞吐 $= 64 \times 1500 \times 8 / 863.5\times10^{-6} = 768000/863.5\mu s = 889$ Mbps

**MAC 效率 $= 889/1201 = 74\%$**。这是 Wi-Fi 典型的 MAC 效率上限（实际 60–75%）。

---

## 4.4 灵敏度与 Rate-vs-Range

### 4.4.1 灵敏度推导

接收灵敏度定义为满足目标 PER（通常 10%）的最小接收功率：

$$
P_{sens} = -174 + 10\log_{10}(B) + NF + \gamma_{req} + \Delta_{impl}
$$

- $-174$ dBm/Hz：290 K 热噪声底 $kT$
- $10\log_{10}B$：$B$ 单位 Hz
- $NF$：接收机噪声系数，典型 4–7 dB
- $\gamma_{req}$：该 MCS 所需 SNR
- $\Delta_{impl}$：实现损耗（信道估计误差、量化、相位噪声），典型 2–4 dB

**算例 4.5**：80 MHz，MCS 9（$\gamma_{req} = 28$ dB），$NF = 5$ dB，$\Delta_{impl} = 3$ dB。

$$
P_{sens} = -174 + 10\log_{10}(80\times10^6) + 5 + 28 + 3
$$

$10\log_{10}(8\times10^7) = 79$ dB

$$
P_{sens} = -174 + 79 + 5 + 28 + 3 = -59 \text{ dBm}
$$

**算例 4.6**：同样条件下 MCS 0（$\gamma_{req} = 3$ dB）：

$$
P_{sens} = -174 + 79 + 5 + 3 + 3 = -84 \text{ dBm}
$$

**MCS 0 与 MCS 9 的灵敏度差 25 dB**。用 $n = 3$ 的路径损耗模型，25 dB 对应距离比 $10^{25/30} = 6.8$ 倍。即：**MCS 9 的可用半径只有 MCS 0 的 1/6.8**。

### 4.4.2 Rate-vs-Range 曲线

**算法**：对每个 MCS，计算灵敏度 → 由路径损耗模型反解最大距离 → 得到 (距离, 速率) 点对 → 连成阶梯曲线。

```python
def rate_vs_range(f_mhz, bw_mhz, Nss, gi_us, tx_pow_dbm,
                  gt_dbi, gr_dbi, n_exp, L0_db, mcs_list):
    """返回 [(distance_m, rate_mbps), ...]"""
    Tsym = 12.8 + gi_us                            # μs
    Nsd  = {20:234, 40:468, 80:980, 160:1960, 320:3920}[bw_mhz]
    curve = []
    for mcs in mcs_list:
        (mod_bits, Rc, snr_req) = MCS_TABLE[mcs]
        Psens = (-174 + 10*np.log10(bw_mhz*1e6) + NF + snr_req + IMPL_LOSS)
        Lmax  = tx_pow_dbm + gt_dbi + gr_dbi - Psens
        # L(d) = L0 + 10*n*log10(d)  ->  d = 10^((Lmax - L0)/(10n))
        d = 10 ** ((Lmax - L0_db) / (10 * n_exp))
        rate = Nsd * mod_bits * Rc * Nss / (Tsym * 1e-6) / 1e6   # Mbps
        curve.append((d, rate))
    return sorted(curve)
```

**典型结果**（80 MHz, 2 SS, 20 dBm, $n = 3$, $L_0 = 40$ dB @1m）：

| MCS | 速率 (Mbps) | 可用距离 (m) |
|---|---|---|
| 0 | 34 | 68 |
| 4 | 206 | 32 |
| 7 | 344 | 21 |
| 9 | 510 | 15 |
| 11 | 637 | 10 |
| 13 | 765 | 5 |

**这条曲线是理解 Wi-Fi 8 动机的基础**：用户在 15 米外用 MCS 4 只有 206 Mbps，而"23 Gbps"的广告与此毫无关系。Wi-Fi 8 的 25% 吞吐提升目标，正是在**中远距离**（MCS 4–8 区间）实现的。

---

## 4.5 新增 MCS 与非均衡调制（引子）

【草案】802.11bn 讨论中的两项 PHY 增强：

1. **新增精细 MCS 档位**：在现有 MCS 之间插入中间档（如更低码率的 16-QAM、256-QAM），使速率阶梯变细。目的是**减少"跳档"造成的吞吐突变**——当 SNR 在两档之间波动时，速率选择更平滑，避免"MCS 4 ↔ MCS 7"的大幅振荡。

2. **UEQM（Unequal Modulation，非均衡调制）**：不同空间流使用**不同调制阶**。第 25 章详细展开。

---

## 本章小结

| 概念 | 结论 |
|---|---|
| BCC | K=7 卷积码，用于 SIG 与低速率 |
| LDPC | QC-LDPC，数据字段主力，比 BCC 增益 1.5–2.5 dB |
| Min-Sum 译码 | 用 $\min$ 替代 $\tanh$ 链式，损失 0.2–0.5 dB |
| 调制阶提升 | 每 +2 bit 需 +6 dB SNR（最小距离减半） |
| EVM 天花板 | $\gamma_{\rm eff} \approx 1/\text{EVM}^2$，功率无法补偿 |
| 速率公式 | $R = N_{sd}N_{bpscs}R_cN_{ss}/T_{sym}$ |
| 灵敏度 | $-174 + 10\log B + NF + \gamma_{req} + \Delta_{impl}$ |
| MAC 效率 | 典型 60–75%，前导与帧间隔是主要开销 |

---

## 习题

**概念题**

1. 为什么 SIG 字段坚持使用 BCC 而数据字段改用 LDPC？从可靠性、延迟、短块性能三个角度回答。
2. 为什么说"提高发射功率无法突破 EVM 造成的 SNR 天花板"？请给出数学说明。
3. MCS 表中相邻档位的 SNR 间隔并不是均匀的（例如 MCS 6→7 只差 2 dB，MCS 11→12 差 4 dB）。这反映了什么设计取舍？

**计算推导题**

4. 推导 MQAM 的符号错误率近似式 $P_s \approx 4(1-1/\sqrt{M})Q\left(\sqrt{\frac{3\log_2 M}{M-1}\gamma_s}\right)$ 中的系数 $\frac{3\log_2 M}{M-1}$。（提示：从最小距离与平均符号能量的关系出发）
5. 计算 320 MHz、4 SS、MCS 13、GI = 0.8 μs 的 PHY 速率。若 MAC 效率 70%，求 TCP 有效吞吐。
6. 某 接入点（Access Point, AP）：20 dBm，天线 3 dBi，终端天线 0 dBi，$NF = 6$ dB，实现损耗 3 dB，20 MHz，MCS 7。求灵敏度与可用距离（$n = 3.2$，$L_0 = 40$ dB @ 1 m）。若改用 80 MHz 同 MCS，可用距离如何变化？说明原因。
7. 证明：在 EVM 受限系统中，$\gamma_{\rm eff} = \gamma/(1 + \gamma\cdot\text{EVM}^2)$ 的极限为 $1/\text{EVM}^2$。若总 EVM 由发射机 EVM $(-36$ dB$)$、相位噪声 $(-38$ dB$)$、量化噪声 $(-40$ dB$)$ 三个独立源构成，求总 EVM（功率域相加）与有效 SNR 上限。

**算法设计题**

8. 设计一个**速率-距离曲线拟合算法**：给定实测的 (距离, MCS, PER) 样本，反推出该场景的路径损耗指数 $n$ 与每个 MCS 的实际 SNR 门限。给出建模、伪代码与复杂度。
9. 设计 LDPC 译码的**早停准则**：除校验通过外，还可以用哪些指标提前终止迭代以省功耗？给出至少两种指标与伪代码。

**开放研究题**

10. 【研究】11bn 提出在 MCS 之间插入精细档位。请分析：(a) 精细档位对链路自适应算法（第 5 章）的探测开销有何影响？(b) 档位越细是否总是越好？给出一个"档位数-系统吞吐"的权衡模型，并指出最优档位数的决定因素。

---

## 答案与解析

**1.** (a) **可靠性**：SIG 字段决定了 PPDU 的带宽、MCS、长度等关键参数，一旦译错整帧报废，且接收端无法请求重传（SIG 都无法解析就不知道是谁发的）。BCC 在短块下无错误平层，可靠性更可预测。(b) **延迟**：Viterbi 译码是前向串行、延迟确定；LDPC 需迭代，迭代次数不确定，译码延迟变化大，不适合作为"必须先解出来才能解后续"的字段。(c) **短块性能**：LDPC 的优势随码块长度增长而显现，SIG 字段仅数十比特，LDPC 优势几乎消失，却要付出更大的译码复杂度。

**2.** 由 $\gamma_{\rm eff} = \frac{\gamma}{1 + \gamma\,\text{EVM}^2}$，令 $\gamma \to \infty$：

$$
\lim_{\gamma\to\infty} \gamma_{\rm eff} = \lim_{\gamma\to\infty} \frac{1}{1/\gamma + \text{EVM}^2} = \frac{1}{\text{EVM}^2}
$$

**物理解释**：EVM 是**乘性**损伤（与信号幅度成比例的失真），提高发射功率时信号与失真**同步放大**，比值不变。这与热噪声（加性、与信号无关）有本质区别——加性噪声可通过提高功率克服，乘性损伤不能。因此高阶调制的覆盖距离由**收发信机的 EVM 预算**决定，而非发射功率。

**3.** 反映了**调制阶提升与码率提升的成本不同**：
- MCS 6→7（64-QAM 3/4 → 5/6）是**同调制阶提码率**，只需约 1–2 dB，但增益也小（速率 +11%）
- MCS 11→12（1024-QAM 5/6 → 4096-QAM 3/4）是**换调制阶**，最小距离减半需约 6 dB，虽然码率下降（5/6→3/4）回收了约 1.2 dB，净需约 4–5 dB，增益 +8%

设计取舍：**低速率区间用密集档位**（因为此区间用户多、信道变化频繁，细档位能精确匹配），**高速率区间档位稀疏**（因为可用窗口窄、用户少，再细分收益低，且增加探测开销）。

**4. 推导**：

MQAM 星座（$M = 2^{2k}$，方形）的坐标为 $\pm d/2, \pm 3d/2, \dots, \pm(\sqrt{M}-1)d/2$，其中 $d$ 为最小距离。

平均符号能量：

$$
E_s = \frac{d^2}{6}(M-1)
$$

（推导：一维 PAM 的 $M_1 = \sqrt{M}$ 个电平 $\pm d/2, \dots$，平均功率 $= \frac{d^2}{12}(M_1^2-1)$；二维方形 正交幅度调制（Quadrature Amplitude Modulation, QAM）为两个独立 PAM 之和：$E_s = 2 \times \frac{d^2}{12}(M-1) = \frac{d^2}{6}(M-1)$。）

故 $d^2 = \frac{6E_s}{M-1}$。

MQAM 的 SER 近似由最近邻决定（union bound 主项）：

$$
P_s \approx 4\left(1 - \frac{1}{\sqrt{M}}\right) Q\left(\frac{d/2}{\sqrt{N_0/2}}\right)
$$

其中 $\frac{d/2}{\sqrt{N_0/2}} = \sqrt{\frac{d^2}{2N_0}} = \sqrt{\frac{6E_s}{2N_0(M-1)}} = \sqrt{\frac{3E_s}{N_0(M-1)}}$

又因为每符号携带 $\log_2 M$ 比特，$E_s = E_b \log_2 M$，$\gamma_s = E_s/N_0$：

$$
P_s \approx 4\left(1 - \frac{1}{\sqrt{M}}\right)Q\left(\sqrt{\frac{3\log_2 M}{M-1}\gamma_s}\right)
$$

系数 $\frac{3\log_2 M}{M-1}$ 出现。$\square$

**5.** 320 MHz：$N_{sd} = 3920$；MCS 13：$N_{bpscs} = 12$，$R_c = 5/6$；$N_{ss} = 4$；$T_{sym} = 13.6$ μs。

$$
R = \frac{3920 \times 12 \times (5/6) \times 4}{13.6\times10^{-6}} = \frac{3920 \times 40}{13.6\times10^{-6}} = \frac{156800}{13.6\times10^{-6}} = 11.53 \text{ Gbps}
$$

MAC 效率 70% → TCP 有效吞吐 $= 8.07$ Gbps。

（注：320 MHz + 4 SS 的峰值常被称为"11.5 Gbps"；8 SS 时为 23.06 Gbps。）

**6.** MCS 7：$\gamma_{req} = 22$ dB。

20 MHz：$10\log_{10}(2\times10^7) = 73$ dB

$$
P_{sens} = -174 + 73 + 6 + 22 + 3 = -70 \text{ dBm}
$$

链路预算：$L_{max} = 20 + 3 + 0 - (-70) = 93$ dB

$$
93 = 40 + 32\log_{10} d \Rightarrow \log_{10} d = 1.656 \Rightarrow d = 45.3 \text{ m}
$$

**80 MHz**：$10\log_{10}(8\times10^7) = 79$ dB，噪声底抬升 6 dB：

$$
P_{sens} = -174 + 79 + 6 + 22 + 3 = -64 \text{ dBm}
$$

$$
L_{max} = 20 + 3 + 0 + 64 = 87 \text{ dB} \Rightarrow 87 = 40 + 32\log_{10}d \Rightarrow d = 29.1 \text{ m}
$$

**距离从 45.3 m 降到 29.1 m（−36%）**。原因：带宽扩大 4 倍，噪声功率（$kTB$）增加 6 dB，而发射功率受 功率谱密度（Power Spectral Density, PSD）限制**不能同步提高**（第 6 章），因此灵敏度恶化 6 dB，覆盖缩小。**这是"宽带宽牺牲覆盖"的根本原因**，也是 Wi-Fi 8 NPCA/DSO（第 26 章）试图缓解的问题之一。

**7.**
(a) 极限：$\gamma_{\rm eff} = \frac{\gamma}{1+\gamma E^2}$，分子分母同除以 $\gamma$：$=\frac{1}{1/\gamma + E^2} \to \frac{1}{E^2}$。$\square$

(b) 三个独立 EVM 源（线性值）：
- $-36$ dB → $10^{-3.6} = 2.512\times10^{-4}$，功率 $6.31\times10^{-8}$
- $-38$ dB → $1.585\times10^{-4}$，功率 $2.51\times10^{-8}$
- $-40$ dB → $1.0\times10^{-4}$，功率 $1.0\times10^{-8}$

总 EVM 功率 $= 6.31 + 2.51 + 1.0 = 9.82\times10^{-8}$（注意：EVM 是**幅度域**量，功率域相加后再开方？——严格地说 EVM 定义为误差功率与信号功率之比的**平方根**，即 $\text{EVM} = \sqrt{P_{err}/P_{sig}}$。因此功率域量是 $\text{EVM}^2$，相加的是 $\text{EVM}^2$。上面的"功率"就是 $\text{EVM}^2$。）

总 $\text{EVM}^2 = 9.82\times10^{-8}$ → $\text{EVM} = 3.13\times10^{-4} = -35.1$ dB

有效 SNR 上限 $= 1/9.82\times10^{-8} = 1.018\times10^{7} = 70.1$ dB

**结论**：最差的那个源（−36 dB）主导了总 EVM；改善其它源收益很小。这是**短板效应**，工程上应优先改善最大的损伤源。

**8. 参考方案**

**建模**：观测 $(d_i, \text{MCS}_i, \text{PER}_i)$。对 MCS $m$，PER 由有效 SNR 决定：

$$
\text{PER} = f_m(\gamma_{rx}), \quad \gamma_{rx} = P_{tx} + G - L(d) - (-174 + 10\log B + NF)
$$

$$
L(d) = L_0 + 10 n \log_{10} d
$$

假设 $f_m$ 已知（由 AWGN 仿真给出 PER-SNR 曲线），则未知参数为 $\{L_0, n, \Delta_{impl,m}\}$（每 MCS 的实现损耗）。

**估计方法**：最大似然——对给定的 $(L_0, n, \{\Delta_m\})$，预测每个样本的 PER，最小化与实测的 KL 散度或加权均方误差：

$$
\min_{L_0, n, \{\Delta_m\}} \sum_i w_i \left(\text{PER}_i^{obs} - f_{m_i}\big(\gamma(d_i; L_0, n, \Delta_{m_i})\big)\right)^2
$$

**伪代码**：

```python
def fit_rate_range(samples, per_curves, tx_pow, gt, gr, bw, nf):
    """samples: [(d, mcs, n_trial, n_err)]"""
    noise_floor = -174 + 10*np.log10(bw*1e6) + nf
    def residual(theta):
        L0, n = theta[0], theta[1]
        deltas = theta[2:]
        res = 0.0
        for (d, mcs, nt, ne) in samples:
            gamma = tx_pow + gt + gr - (L0 + 10*n*np.log10(d)) \
                    - noise_floor - deltas[mcs]
            per_pred = per_curves[mcs](gamma)
            per_obs = ne / nt
            res += nt * (per_obs - per_pred)**2 / max(per_pred*(1-per_pred), 1e-6)
        return res
    theta0 = np.concatenate([[40.0, 3.0], np.zeros(N_MCS)])
    return scipy.optimize.minimize(residual, theta0, method='L-BFGS-B')
```

**复杂度**：$O(N_{iter} \cdot N_{sample})$，典型数百次迭代、数千样本，秒级完成。
**要点**：① 必须用**加权**残差（$n_{trial}$ 作为权重，且除以二项方差）；② PER 曲线的插值必须有物理约束（单调递减、取值 [0,1]）；③ 样本需覆盖足够远的距离范围，否则 $n$ 不可辨识；④ 评分要点：能否指出 $L_0$ 与 $\Delta_{impl}$ 存在**参数不可辨识性**（两者在回归中高度相关，需固定其一或用 MCS 0 的样本单独标定 $L_0$）。

**9. 参考方案**：

除 $\mathbf{H}\hat{\mathbf{x}} = 0$ 外，可用的早停指标：

(a) **LLR 幅度均值**：$\bar{L} = \frac{1}{n}\sum_j |L_{post,j}|$。若 $\bar{L}$ 超过阈值（如 15），说明判决极可靠，可停止。
(b) **校验方程满足数**：不要求全部满足，若满足数 > $0.98m$ 且连续 2 轮不再改善，则停止（避免在无解时空转 20 轮）。
(c) **消息熵**：变量节点消息的符号翻转率，若连续两轮翻转率 < 0.1%，认为已收敛。
(d) **硬判决稳定性**：连续 3 轮硬判决结果相同。

```python
def ldpc_decoder_early_stop(Lch, H, max_iter=20, L_thr=15.0,
                            sat_ratio=0.98, stab_rounds=3):
    prev_bits = None; stable = 0
    for it in range(max_iter):
        Lpost, n_sat = ldpc_iteration(Lch, H)          # 一轮 BP
        bits = (Lpost < 0).astype(int)
        if n_sat == H.shape[0]:                        # 准则1: 校验全通过
            return bits, it+1, 'parity'
        if np.mean(np.abs(Lpost)) > L_thr:             # 准则2: LLR 足够大
            return bits, it+1, 'llr'
        if n_sat > sat_ratio * H.shape[0]:             # 准则3: 近似满足
            return bits, it+1, 'near_parity'
        if prev_bits is not None and np.array_equal(bits, prev_bits):
            stable += 1
            if stable >= stab_rounds:                  # 准则4: 判决稳定
                return bits, it+1, 'stable'
        else:
            stable = 0
        prev_bits = bits.copy()
    return (Lpost<0).astype(int), max_iter, 'max_iter'
```

**功耗收益**：实测典型场景下可使平均迭代次数从 20 降至 6–8，译码功耗降低 60–70%，代价是 0.05–0.1 dB 性能损失（主要来自准则 2/3 的误判）。
**评分要点**：① 必须给出各准则的**误判风险**与阈值标定方法；② 应说明准则的**优先级**（先查校验，再查 LLR）；③ 功耗模型要区分"每轮迭代能耗"与"控制逻辑能耗"。

**10. 参考思路**：

(a) **探测开销**：链路自适应（第 5 章 Minstrel）需要周期性"探测"未使用的 MCS 以更新统计。档位从 14 增至 $M$ 档，探测一轮所需时间 $\propto M$，在快变信道中可能出现"还没探完信道就变了"的问题。定量：若每档需 10 个样本、每样本 5 ms，14 档需 0.7 s，24 档需 1.2 s；而步行场景 $T_c = 17$ ms，**探测永远跟不上**——这说明精细档位必须配合**基于 SNR 的模型驱动自适应**（用 SNR 查表推断所有档位的性能，而非逐个探测）。

(b) **权衡模型**：

$$
\text{净吞吐} = \underbrace{\mathbb{E}[\text{实际速率}]}_{\text{档位越细越高}} - \underbrace{C_{probe}\cdot M}_{\text{探测开销}} - \underbrace{C_{sig}\cdot \lceil\log_2 M\rceil}_{\text{信令开销}}
$$

第一项：设 SNR 在 $[\gamma_{min}, \gamma_{max}]$ 均匀分布，档位边界最优放置时，量化损失 $\approx \frac{(\Delta\gamma)^2}{24}\cdot\frac{d^2R}{d\gamma^2}$，随 $M$ 按 $1/M^2$ 下降。

$$
\mathbb{E}[R] \approx R_{ideal} - \frac{K}{M^2}
$$

最优：$ \frac{d}{dM}\left(R_{ideal} - \frac{K}{M^2} - C_{probe}M\right) = \frac{2K}{M^3} - C_{probe} = 0$

$$
M_{opt} = \left(\frac{2K}{C_{probe}}\right)^{1/3}
$$

**决定因素**：① 信道变化速度（快 → $C_{probe}$ 大 → $M_{opt}$ 小）；② SNR 分布范围（宽 → $K$ 大 → $M_{opt}$ 大）；③ 速率- SNR 曲线的曲率（高阶调制区曲率大，需要更细档位）。
**结论**：档位并非越细越好，且**不同 SNR 区间应有不同密度**（高速率区间更细），这与习题 3 的结论一致。

---

## 延伸阅读

- [IEEE] IEEE 802.11-2024, Clause 17.3.5.5（BCC）、19.3.11.7（LDPC）、Clause 36（Extremely High Throughput, EHT MCS）
- [ACM/IEEE] Richardson & Urbanke, *Modern Coding Theory*, Cambridge University Press, 2008
- [IEEE] IEEE 802.11-23/XXXX，EHT MCS 表与 SNR 门限提案
- [TUT] IEEE 802.11bn 草案中关于新增 MCS 与 UEQM 的讨论（见第 25 章）
