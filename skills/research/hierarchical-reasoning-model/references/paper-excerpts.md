# HRM 論文核心摘錄

> 來源: arxiv.org/abs/2506.21734 (Wang et al. 2025, Sapient Intelligence)
> 擷取日期: 2026-06-09

## 核心問題

> "The fixed depth of standard Transformers places them in computational complexity classes such as AC⁰ or TC⁰, preventing them from solving problems that require polynomial time. LLMs are not Turing-complete."

> "CoT for reasoning is a crutch, not a satisfactory solution. It relies on brittle, human-defined decompositions where a single misstep or a misorder of the steps can derail the reasoning process entirely."

## 層級收斂

> "During each cycle, the L-module exhibits stable convergence to a local equilibrium. This equilibrium, however, depends on the high-level state zH supplied during that cycle. After completing the T steps, the H-module incorporates the sub-computation's outcome and performs its own update. This zH update establishes a fresh context for the L-module, essentially 'restarting' its computational path and initiating a new convergence phase toward a different local equilibrium."

## 1-step Gradient

> "If a recurrent neural network converges to a fixed point, we can avoid unrolling its state sequence by applying backpropagation in a single step at that equilibrium point."

> "Given that each module only needs to back-propagate errors through its most recent local synaptic activity, this approach aligns well with the perspective that cortical credit assignment relies on short-range, temporally local mechanisms."

## Deep Supervision

> "Inspired by the principle that periodic neural oscillations regulate when learning occurs in the brain, we incorporate a deep supervision mechanism."

## ACT

> "The brain dynamically alternates between automatic thinking ('System 1') and deliberate reasoning ('System 2')."

## 腦科學對應

> "A region's functional repertoire—its ability to handle diverse and complex tasks—is closely linked to the dimensionality of its neural representations."

> "The high-to-low PR ratio in HRM (zH/zL ≈ 2.98) closely matches that measured in the mouse cortex (≈ 2.25)."

> "HRM autonomously discovers an organizational principle that is thought to be fundamental for achieving robust and flexible reasoning in biological systems."

## 關鍵數據

- 27M 參數, 1000 樣本, 無預訓練, 無 CoT
- ARC-AGI-1: 40.3% (vs o3-mini-high 34.5%)
- Sudoku-Extreme: 55.0% (vs CoT models 0%)
- Maze-Hard: 74.5% (vs CoT models 0%)
- N=2, T=2, M_max=2-8
