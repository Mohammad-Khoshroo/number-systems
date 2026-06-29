# Logarithmic Number Systems (LNS) for Deep Neural Network (DNN) Architectures

## Introduction

The use of Logarithmic Number Systems (LNS) for neural computations was first proposed in the late 1990s. Since then, this approach has gained increasing popularity for efficient hardware implementation of deep neural networks.

### Main Advantages of LNS

The primary advantage of using LNS lies in simplifying the implementation of computationally expensive operations required for inference and/or training of deep neural networks. Additionally, representing data in LNS format enables reducing the number of bits needed to achieve the same DNN accuracy compared to conventional number systems (such as standard floating-point).

## Number Representation in LNS

### Data Structure

A real number $n$ in the logarithmic number system is represented as follows:

- **Logarithm of the absolute value** with base $a$: $\tilde{n} = \log_a(|n|)$
- **A sign bit**: $s_n$ (to determine whether the number is positive or negative)

The logarithmic value ($\tilde{n}$) is represented using **two's complement fixed-point** format.

### Choice of Logarithmic Base

The logarithmic base ($a$) is typically chosen to be 2 for simpler hardware implementation. This is because in digital hardware, working with base 2 is much more efficient. Base-2 logarithm (similar to bit-shift operations in programming) can be implemented with very simple logic structures, eliminating the need for complex mathematical circuits to compute logarithms.

## The Challenge of Representing Zero in LNS

### Nature of the Problem

One of the fundamental challenges of LNS is representing zero:

$$\log_2(0) = -\infty$$

Since the logarithm of zero is undefined, it cannot be directly represented in fixed-point format. This issue is critical in neural networks because:
- Weights may be adjusted toward zero (pruning)
- ReLU activations produce zero outputs
- Dropout operations temporarily convert values to zero

### Common Solutions

#### 1. Reserved Value

Using a specific bit pattern as a **logical zero flag**:

Example with 8 bits:
Logical zero = 10000000 (smallest representable negative value)


**Hardware Implementation:**
- An additional `zero_flag` bit is added to each number
- Computing units check this flag before each operation
- If the flag is active, they bypass the operation

**Advantages:**
- Exact representation of zero
- Correct mathematical behavior in operations

**Disadvantages:**
- Storage overhead (one additional bit per number)
- Control logic complexity

#### 2. Saturation to Smallest Value

Using the smallest representable value instead of true zero:

$$\tilde{n}_{\text{zero}} = -2^{I-1} \implies n_{\text{approx}} = 2^{-2^{I-1}}$$

Example with $I=5$:
$$n_{\text{approx}} = 2^{-16} \approx 0.000015$$

**Advantages:**
- No storage overhead
- Very simple implementation

**Disadvantages:**
- Zero is approximate, not exact
- In deep networks, error may accumulate

**Application in DNNs:**
This method is used in most LNS implementations for DNNs because:
- Neural networks are inherently noise-tolerant
- The error from this approximation is negligible compared to other errors (quantization, dropout)
- Network performance is typically unaffected

#### 3. Addition Logic Management (Bypass Logic)

Implementing conditional paths in the LNS addition unit:

If n₁ = 0:
    output = n₂
If n₂ = 0:
    output = n₁
Otherwise:
    Standard LNS addition computation


**Advantages:**
- Correct mathematical behavior
- Prevents error accumulation

**Disadvantages:**
- Control logic complexity
- Increased critical path latency

### Comparison with Conventional Systems

**Zero representation in different systems:**

| System | Zero Representation | Complexity |
|--------|-------------------|-----------|
| Fixed-point | $00...0$ | No overhead |
| Floating-point (IEEE 754) | Zero symbol with all zero bits | Standard logic |
| LNS | Requires special solution | Additional complexity |

This is one of the **design costs** of LNS that must be evaluated against its advantages (elimination of multipliers).

## Core Operations in LNS Domain

### Multiplication in LNS

The primary operation in neural networks that is dramatically simplified using LNS is **multiplication**, which is transformed into **linear addition** (fixed-point).

The LNS product (denoted as $\tilde{p}$) for two real numbers $n_1$ and $n_2$ is computed as:

$$\tilde{p} = \tilde{n}_1 \odot \tilde{n}_2 = \log_2(|n_1| \times |n_2|) = \tilde{n}_1 + \tilde{n}_2$$

**Product sign bit:**

$$s_{\tilde{p}} = s_{n_1} \text{ XOR } s_{n_2}$$

This means hardware that previously required a complex multiplier unit now only needs an adder unit, which is much smaller and faster. For determining the result's sign, a very simple logic gate called XOR is used.

### Challenge of Addition and Activation Functions

The challenging part is implementing **LNS addition** and **LNS activation functions**. Unlike multiplication which became addition, the addition operation in logarithmic space is complex:

$$\log(A + B) = \log(A) + \log\left(1 + 2^{\log B - \log A}\right)$$

This means to perform a simple addition, the hardware must perform exponentiation ($2^x$) and logarithm operations again, which is very complex and costly.

### Addition Solution in LNS

The LNS addition formula is simplified as follows:

- If signs are the same ($s_{n1} = s_{n2}$):

$$s\tilde{u}m = \max(\tilde{n}_1, \tilde{n}_2) + \log_2(1 + 2^{-|\tilde{n}_1 - \tilde{n}_2|})$$

- If signs are different ($s_{n1} \neq s_{n2}$):

$$s\tilde{u}m = \max(\tilde{n}_1, \tilde{n}_2) + \log_2(1 - 2^{-|\tilde{n}_1 - \tilde{n}_2|})$$

The term $\Delta_{\pm} = \log_2(1 \pm 2^{-|\tilde{n}_1 - \tilde{n}_2|})$ is approximated using **Look-Up Tables (LUTs)** or **bit shifts**.

### Bit-Shift Implementation

For bit-shift implementation, the following approximation is used:

$$\log_2(1 + x) \approx x \quad \text{for } 0 < x < 1$$

Therefore:

$$\log_2(1 \pm 2^{-|\tilde{n}_1 - \tilde{n}_2|}) \approx \pm \mathbf{BS}(1, -|\tilde{n}_1 - \tilde{n}_2|)$$

where $\mathbf{BS}(b; d) = b \times 2^d$ means shifting the binary representation bits of $b$ by $|d|$ positions.

**Advantages of this method:**
- Extremely fast: shift operation is one of the fastest and lowest-cost operations in computer architecture
- Memory elimination: no need for memory to store tables; everything is done with simple logic gates

### Implementing $2^x$ for Fractional Exponentiation

One of the key operations in LNS is converting from logarithmic to linear space, which requires computing $2^{\tilde{p}}$.

#### Separating Exponent into Integer and Fractional Parts

Any logarithmic number $\tilde{p}$ can be split into two parts:

$$\tilde{p} = \underbrace{i}_{\text{integer part}} + \underbrace{f}_{\text{fractional part}}, \quad 0 \le f < 1$$

Therefore:

$$2^{\tilde{p}} = 2^{i+f} = 2^{i} \cdot 2^{f}$$

#### Implementing $2^i$ (Integer Part)

Computing $2^i$ is equivalent to **bit shifting**:

- If $i > 0$: left shift by $i$ bits
- If $i < 0$: right shift by $|i|$ bits
- If $i = 0$: no shift

Example:
Number: 00001011 (11 in decimal)
2³ × number = 01011000 (88 in decimal)
Left shift by 3 bits


This operation has **zero computational cost**, just rewiring in hardware.

#### Implementing $2^f$ (Fractional Part)

This is the real challenge because $0 \le f < 1$ and the result is a number between 1 and 2 that cannot be implemented with simple shifting.

**Method 1: Mitchell's Approximation**

The simplest method uses linear approximation:

$$2^f \approx 1 + f$$

**Examples:**
- $2^{0.5} \approx 1.5$ (actual value: $1.414$, error: $6\%$)
- $2^{0.25} \approx 1.25$ (actual value: $1.189$, error: $5\%$)

**Implementation:**
If f = 0.b₁b₂b₃b₄ (binary)
Then 2^f ≈ 1.b₁b₂b₃b₄


Meaning we directly use the fractional bits as the mantissa.

**Advantages:**
- No computation required
- Fastest possible method

**Disadvantages:**
- Relative error up to 11%
- Maximum error occurs at $f=1$

**Method 2: Look-Up Table (LUT)**

Pre-compute and store $2^f$ values for all possible values of $f$.

**Example with 4 fractional bits:**
f      | 2^f (actual)
-------|-------------
0.0000 | 1.0000
0.0001 | 1.0045
0.0010 | 1.0090
...
0.1111 | 1.9307


Number of table entries: $2^F$ where $F$ is the number of fractional bits.

**Advantages:**
- High accuracy (limited only by quantization)
- Constant access speed

**Disadvantages:**
- Memory requirement: with $F=8$ bits, needs $256$ entries
- For high precision, required memory grows rapidly

**Method 3: Piece-wise Linear Approximation**

Divide the $2^f$ curve into several line segments:

$$2^f \approx m_k \cdot f + c_k \quad \text{for } f \in [f_k, f_{k+1})$$

**Example with 2 segments:**
Segment 1 (0 ≤ f < 0.5): 2^f ≈ 1.4f + 1.0
Segment 2 (0.5 ≤ f < 1): 2^f ≈ 1.8f + 0.8


**Implementation:**
1. Use high bits of $f$ to select segment
2. Read slope coefficient $m_k$ and intercept $c_k$ for that segment
3. Compute $m_k \cdot f + c_k$ with one multiply and one add

**Advantages:**
- Lower error than Mitchell (typically under 3%)
- Less memory than full LUT

**Disadvantages:**
- Requires a small multiplier
- Medium complexity

#### Method Comparison

| Method | Maximum Error | Memory (F=8) | Computational Complexity |
|--------|--------------|--------------|-------------------------|
| Mitchell | ~11% | 0 bytes | Zero (wiring only) |
| Full LUT | <0.5% | 256 entries | One memory read |
| Piece-wise (4 segments) | ~3% | 8 entries | One multiply + one add |

#### Relationship to Logarithm Conversion

This process is exactly the **inverse** of logarithm conversion:

**Conversion to logarithm:**
$$\log_2(1 + x) \approx x$$

**Conversion from logarithm:**
$$2^f \approx 1 + f$$

Both use **the same linear approximation**, just in opposite directions. This symmetry causes the errors of these two operations to partially cancel each other.

### Activation Functions

For more complex activation functions such as **Sigmoid**, **tanh**, and **Softmax**, more efficient hardware is achieved if these functions are approximated using **piece-wise approximations**. These approximations can be implemented using **combinational logic**.

This approximation becomes an additional source of non-linearity and imposes a small computational burden on the implemented DNN architecture's performance. This is particularly true when these functions are used during the **training process**, which is inherently noisy, and the neural network typically adapts itself to these differences.

#### Implementing Leaky-ReLU in LNS Space

Some activation functions can be directly transferred to the logarithmic domain using LNS operations. One of the most elegant examples is **Leaky-ReLU** implementation.

**Definition in linear space:**

$$LReLU(n|\alpha) = \begin{cases} \alpha n, & n < 0 \\ n, & n \ge 0 \end{cases}$$

**Representation in LNS space:**

$$\tilde{LReLU}((\tilde{n}, s_n)|\alpha) = \begin{cases} (\tilde{n} + \tilde{\alpha}, s_a = s_n), & s_n = 1 \\ (\tilde{n}, s_a = s_n), & s_n = 0 \end{cases}$$

**Advantages of LNS implementation for Leaky-ReLU:**

- **Positive part ($s_n = 0$)**: number remains unchanged
- **Negative part ($s_n = 1$)**: multiplication operation ($\alpha \times n$) becomes a **simple addition** ($\tilde{n} + \tilde{\alpha}$)
- **Zero cost**: in hardware, implementation is done only by checking the sign bit and a small adder
- **Integration**: no need for conversion between different spaces

#### Implementing More Complex Functions

**Piece-wise approximation method:**

Instead of exact computation of curved and complex functions like Sigmoid, the curve is divided into several small straight lines. For each numerical range, a simple linear equation is defined.

**Advantages:**
- Elimination of heavy exponential mathematical computations ($e^x$)
- Direct implementation with logic gates (AND, OR, XOR)
- Elimination of need for complex processors or large memories
- Very high response speed

**Impact on performance:**

- **Useful error**: approximations introduce some error that itself contributes to non-linearity
- **Noise tolerance in training**: the training process is inherently noisy due to its probabilistic nature and use of gradient descent, so the network typically adapts itself to approximate activation functions
- Final accuracy does not show significant degradation

## Different Approaches to Using LNS in DNNs

Existing proposals for LNS-based neural networks include:

### 1. End-to-End LNS

Using LNS for the entire DNN architecture from start to finish:

- No conversion from or to conventional systems occurs
- Assumption is that inputs (dataset) and weights are injected into the neural network in LNS format
- This task is typically performed offline and has no overhead on the implemented architecture

**Applications:**
- **Convolutional Neural Networks (CNNs)**: for image and video processing
- **Recurrent Neural Networks (RNNs)**: for text and audio processing (time-series data)

**Implementation conditions:**

When LNS is used to represent all DNN data, all operations required for training and/or inference must be implemented in the LNS domain:

- Multiplication is performed using fixed-point addition
- Other operations (addition and activation functions) must be approximated

**Performance results:**

The proposed approximation techniques cause negligible degradation in the performance of implemented architectures. Classification accuracy reduction in LNS-based CNN architectures has been reported as **less than 1%**.

**Hardware optimization (LSTM case study):**

- In 9-bit design, chip area is **saved by up to 36%**
- As the number of bits increases, area savings decrease; this is due to the need for larger look-up tables for LNS addition approximation

**Conclusion:** LNS is the best choice for **low-power** and **low-precision** systems, which are critical in edge computing and mobile devices.

### 2. Using LNS-based Multipliers

In this approach, only units that perform multiplication are replaced with LNS units. Since a large portion of DNN computational volume relates to multiplication, this change alone can dramatically increase overall system efficiency without needing to change the entire network structure or how data is stored in memory.

#### Mitchell's Multiplier

Any number can be written as $n = 2^k(1+x)$ where:
- $k$ is the power-of-2 part
- $x$ is the fractional/remainder part

**Mitchell's core approximation:**

$$\log_2(1+x) \approx x \quad \text{for } 0 < x < 1$$

Therefore:

$$\log_2(n_1 \times n_2) \approx k_1 + k_2 + x_1 + x_2$$

**Final formula:**

$$n_1 \times n_2 \approx \begin{cases} 2^{k_1+k_2}(1+x_1+x_2), & x_1+x_2<1 \\ 2^{k_1+k_2+1}(x_1+x_2), & x_1+x_2 \ge 1 \end{cases}$$

**Results:**
- Error created by Mitchell approximation is relatively high (up to 11%)
- This multiplier showed no accuracy reduction for 32-bit CNN architecture
- Compared to conventional multipliers with the same number of bits, it is **26.8% more efficient in power consumption**

#### Efficient Mitchell's Multiplier

A **truncated-operand** approach has been proposed where instead of using complete operands, they are truncated and only the $w$ most significant bits are used to compute the approximate product.

**Results with w = 8:**
- Compared to an exact 32-bit FXP multiplier and regular Mitchell multiplier, saves up to **88% in area** and **56% in power**
- Additional error from this bit truncation caused only **0.2% accuracy reduction** for the ImageNet dataset

#### Double-sided Error Multiplier

In Mitchell's method, approximation always underestimated the actual value (one-sided error). In the double-sided method, error is distributed between positive and negative values, which cancel each other in successive additions.

**Double-sided representation:**

$$n = 2^{k+1}(1-y)$$

**Logarithm approximation:**

$$\log_2(n) \approx \begin{cases} k + x, & n - 2^k < 2^{k+1} - n \\ k + 1 - y, & \text{otherwise} \end{cases}$$

**Advantage:** Since some error is positive and some negative, when these numbers are added together in neural network layers, the average error approaches zero. This means computation accuracy increases significantly without greatly increasing complexity.

### 3. Logarithmic Quantization

In this method, the computational hardware structure is not necessarily logarithmic, but rather the **data** (network weights or inputs) are compressed (quantized) logarithmically.

**Advantages:**
- When network weights are stored with fewer bits (e.g., 4 or 8 bits instead of 32 bits) logarithmically, model size becomes very small
- Large AI models can fit in small memories
- Data transfer speed from memory to processor increases dramatically

## Quantization Error and Non-uniform Distribution

### Why is LNS Suitable for DNNs?

One of the main reasons for LNS success in neural networks is **alignment with natural data distribution**.

#### Distribution of Weights and Activations in DNNs

Empirical observations show that in most trained neural networks:

- Weights have a distribution close to **normal** or **log-normal** centered around zero
- Activations also primarily take small values close to zero
- The number of large values (far from zero) is very small

In other words, the majority of data is concentrated in a small range near zero, and only a small percentage of data has large values.

#### Alignment of LNS Resolution with Data Distribution

The key feature of LNS is its **non-uniform resolution**:

$$\Delta n = n \times \ln(2) \times 2^{-F}$$

This formula shows that the distance between consecutive representable numbers is **proportional to the number itself**:

- **For small numbers (near zero)**: $\Delta n$ is small $\Rightarrow$ **high resolution**
- **For large numbers**: $\Delta n$ is large $\Rightarrow$ **low resolution**

**Comparison with fixed-point:**

| Feature | Fixed-point | LNS |
|---------|------------|-----|
| Distance between representation points | Constant (uniform) | Proportional to number |
| Resolution near zero | Low | **High** |
| Resolution for large numbers | High | Low |
| Alignment with DNN distribution | Weak | **Excellent** |

#### Conclusion from Alignment

Since DNN data is primarily concentrated near zero and LNS provides high resolution precisely in this region, **overall quantization error** is significantly reduced. This means the same accuracy as conventional systems can be achieved with fewer bits.

This natural alignment is one of the strongest arguments for using LNS in neural network hardware.

## Conclusion

Logarithmic Number Systems (LNS) are a powerful approach for optimizing deep neural network hardware. By converting expensive multiplication operations to simple addition, these systems enable designing smaller, faster, and lower-power chips.

**Key advantages:**
- Dramatic energy consumption reduction (up to 56%)
- Chip area reduction (up to 88%)
- Maintaining acceptable accuracy (less than 1% degradation)
- Enabling complex model execution on resource-constrained devices

**Main challenges:**
- Complexity of addition operations in logarithmic space
- Need for approximating activation functions
- Reduced efficiency at very high precisions

The choice between different approaches (End-to-End, Multipliers, or Quantization) depends on specific application requirements, but all demonstrate that LNS is an effective solution for hardware optimization in artificial intelligence.

---

## References

1. Arnold et al., "On the cost economy of log arithmetic for backpropagation training on SIMD processors," 1997
2. Miyashita et al., "Convolutional neural networks using logarithmic data representation," 2016
3. Kouretas et al., "Logarithmic number system for deep learning," 2018
4. Sanyal et al., "Neural network training with approximate logarithmic computations," 2020
5. S. A. Alam, J. Garland, and D. Gregg, "Low-precision logarithmic number systems: Beyond base-2," ACM Transactions on Architecture and Code Optimization (TACO), vol. 18, no. 4, pp. 1–25, 2021
6. J. N. Mitchell, "Computer multiplication and division using binary logarithms," IRE Transactions on Electronic Computers, 1962
7. Kim et al., "Efficient Mitchell's approximate log multipliers...," 2018