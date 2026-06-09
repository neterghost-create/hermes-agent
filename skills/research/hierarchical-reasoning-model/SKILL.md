---
name: hierarchical-reasoning-model
description: "HRM (Hierarchical Reasoning Model) — 腦啟發式分層推理架構，27M 參數、1000 樣本即可超越 CoT 模型。"
version: 1.0.0
author: Guan Wang et al. (Sapient Intelligence)
license: arXiv preprint 2506.21734
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [reasoning, brain-inspired, hierarchical, latent-reasoning, architecture, ARC, Sudoku]
    category: research
    related_skills: [self-improvement-protocol]
---

# Hierarchical Reasoning Model (HRM)

> **Paper:** [Hierarchical Reasoning Model](https://arxiv.org/abs/2506.21734) — Guan Wang, Jin Li, Yuhao Sun et al. (Sapient Intelligence, 2025)
> **Code:** https://github.com/sapientinc/HRM

## 概要

> **論文摘錄:** `references/paper-excerpts.md` — 核心引文 + 關鍵數據

HRM 是一種**腦啟發式遞迴架構**，通過兩個耦合的遞迴模組（高層 + 低層）實現深度推理，無需預訓練或 CoT 數據。僅用 **27M 參數 + 1000 樣本**，在 ARC-AGI、Sudoku-Extreme、Maze-Hard 等任務上超越了 o3-mini-high、DeepSeek R1、Claude 3.7 等大模型。

## 核心設計原則（靈感來自大腦）

1. **層級處理 (Hierarchical Processing)** — 高層皮質區域整合長時間尺度信息形成抽象表徵，低層區域處理即時細節
2. **時間分離 (Temporal Separation)** — 不同層級以不同時間尺度運作（如慢 θ 波 4-8Hz vs 快 γ 波 30-100Hz）
3. **遞迴連接 (Recurrent Connectivity)** — 反覆迴路實現迭代精煉，避免 BPTT 的生物不合理性

## 架構

```
HRM = 4 個可學習組件:
├── fI(·; θI) — 輸入嵌入層
├── fL(·; θL) — 低層遞迴模組 (快速、細節計算)
├── fH(·; θH) — 高層遞迴模組 (慢速、抽象規劃)
└── fO(·; θO) — 輸出頭 (softmax/stablemax)
```

### 前向傳播流程

```
輸入 x → x̃ = fI(x)                         # 嵌入
對每個高層週期 k = 1..N:
  對每個低層步驟 i = 1..T:
    zL[i] = fL(zL[i-1], zH[k-1], x̃)        # 低層更新 (每步)
  zH[k] = fH(zH[k-1], zL[T])                # 高層更新 (每週期一次)
輸出 ŷ = fO(zH[N×T])                         # 從高層狀態預測
```

- **N** = 高層週期數
- **T** = 每週期的低層步數
- 總前向步數 = N × T
- 兩個模組都用 **encoder-only Transformer block** (Llama 風格: RoPE, GLU, RMSNorm)

### 層級收斂 (Hierarchical Convergence)

標準 RNN 的致命缺陷：**過早收斂** — 隱藏狀態快速趨向不動點，後續計算步驟失效。

HRM 的解法：
- 低層在每個週期內穩定收斂到**局部平衡點**
- 高層更新後**重置低層**的計算軌跡，啟動新的收斂階段
- 實現 N×T 步的**有效計算深度**，而非標準 RNN 的 T 步
- 高層保持高計算活躍度（大 forward residual），低層週期性收斂後被重置（殘差尖峰）

## 訓練方法

### 1. 單步梯度近似 (One-Step Gradient)

**核心突破**：避免 BPTT 的 O(T) 記憶體開銷。

```
梯度路徑: output_head → H-module 最終狀態 → L-module 最終狀態 → input embedding
記憶體: O(1) vs BPTT 的 O(T)
```

理論基礎：Deep Equilibrium Models (DEQ) + 隱函數定理 (IFT)。近似 (I - J_F)^{-1} ≈ I（Neumann 級數第一項）。

### 2. 深度監督 (Deep Supervision)

受大腦神經振盪調節學習時機的啟發：

```python
# PyTseudocode
for x, y_true in train_dataloader:
    z = z_init
    for step in range(N_supervision):
        z, y_hat = hrm(z, x)          # 前向傳播
        loss = cross_entropy(y_hat, y_true)
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        z = z.detach()                 # 關鍵：截斷梯度圖
```

- 每個監督段(segment) = 一次前向傳播
- **z.detach()** 讓隱藏狀態跨段傳遞但梯度不回傳 → 1-step 近似
- 提供更頻繁的反饋，充當正則化機制

### 3. 自適應計算時間 (ACT)

受 Kahneman「快思慢想」啟發，用 Q-learning 動態決定計算量：

- **Q-head**: 預測 halt/continue 的 Q 值
- **halt** 條件: Q̂_halt > Q̂_continue 且已達最少段數 M_min
- **獎勵**: halt → 1{ŷ=y} (預測正確=1), continue → 0
- **穩定性**: Post-Norm (RMSNorm) + AdamW (參數有界) 保證 Q-learning 收斂

**推理時擴展 (Inference-time Scaling)**：訓練時 M_max=8，推理時可增到 M_max=16 繼續提升準確率，無需重新訓練。

## 實驗結果

### 性能對比 (~1000 訓練樣本，無預訓練，無 CoT)

| 任務 | HRM (27M) | o3-mini-high | Claude 3.7 8K | DeepSeek R1 |
|------|-----------|-------------|---------------|-------------|
| ARC-AGI-1 | **40.3%** | 34.5% | 21.2% | — |
| ARC-AGI-2 | **3.0%** | 1.3% | — | — |
| Sudoku-Extreme (9×9) | **55.0%** | 0.0% | 0.0% | 0.0% |
| Maze-Hard (30×30) | **74.5%** | 0.0% | 0.0% | 0.0% |

### 關鍵觀察

- **CoT 在 Sudoku/Maze 完全失敗** (0%)，HRM 近乎完美
- **增加寬度無用，增加深度是關鍵** — Sudoku 上 Transformer 寬度翻倍無提升，深度增加才有幫助
- **標準架構深度飽和** — 標準/遞迴 Transformer 在 >64 層後不再受益，HRM 持續受益
- **1000 樣本即可學習** — 數據效率極高

## 腦科學對應

### 維度層級 (Dimensionality Hierarchy)

用 **Participation Ratio (PR)** 衡量表徵維度：

- **高層模組 z_H**: PR = 89.95（高維、靈活、認知彈性）
- **低層模組 z_L**: PR = 30.22（低維、專門化）
- **PR 比值 z_H/z_L ≈ 2.98**，與小鼠皮質測量值 (~2.25) 高度吻合
- 未訓練網路無此層級分離 → **是訓練中湧現的，非架構假象**

這與前額葉皮質 (PFC) 的高維表徵一致，被認為是複雜推理的關鍵。

## 推理行為可視化

| 任務 | 推理策略 |
|------|---------|
| **Maze** | 先同時探索多條路徑 → 消除死路 → 構建初步解 → 多次精煉 |
| **Sudoku** | 深度優先搜索風格：探索候選解 → 碰到死路回溯 |
| **ARC** | 類似爬山優化：增量調整 → 迭代改進直到收斂 |

模型能**自動適應不同任務選擇不同策略**。

## 計算普適性

HRM 具備 **Turing 完備性**（給定足夠記憶體和時間），克服了標準 Transformer 的 AC⁰/TC⁰ 計算複雜度限制。配合 ACT，接近實用的 Turing 完備計算。

## 與現有方法的關係

| 方法 | 問題 |
|------|------|
| **CoT** | 脆弱的任務分解、大量數據需求、高延遲 |
| **RL + CoT** | 不穩定、數據低效、解鎖已有能力而非發現新機制 |
| **標準 RNN** | 過早收斂、BPTT 記憶體 O(T) |
| **Linear Attention** | 替代注意力但仍是固定深度，仍需 CoT |
| **HRM** | ✅ 深度推理 + 穩定訓練 + O(1) 記憶體 + 數據高效 |

## 關鍵超參數

| 參數 | 值 | 說明 |
|------|-----|------|
| N (高層週期) | 2 | 高層更新次數 |
| T (低層步數) | 2 | 每週期低層更新次數 |
| M_max (ACT) | 2-8 | 最大監督段數 |
| M_min | 隨機 ε-greedy | 最少段數 |
| 參數量 | 27M | 兩個模組各 ~12M + 嵌入/輸出 |
| 訓練樣本 | ~1000 | 無預訓練 |
| 優化器 | Adam-atan2 | Adam 的 scale-invariant 變體 |
| 初始化 | truncated LeCun Normal | σ=1, trunc=2 |
| 損失 | Seq2Seq + ACT Q-learning | 混合損失 |

## 應用場景

1. **符號推理任務** — Sudoku、迷宮、規劃問題
2. **少樣本歸納推理** — ARC-AGI 類任務
3. **需要深度搜索/回溯的問題** — 演算法學習
4. **邊緣部署** — 27M 參數適合資源受限環境
5. **替代 CoT** — 不需要生成中間 token，低延遲

## 局限性與未來方向

- ARC-AGI-2 上仍只有 3%，距人類水平甚遠
- 腦科學對應是**相關性**而非因果性
- 目前僅在離散符號任務上驗證，NLP/對話場景未測試
- 可探索方向：整合 hierarchical memory、linear attention、更複雜的合併機制（目前用簡單加法）

## 與 Self-Improvement Protocol 的同構映射

HRM 的架構設計原則可以直接應用到 Agent 的元學習系統 (self-improvement-protocol)：

| HRM | Agent 元學習 | 進化方向 |
|-----|------------|---------|
| 高層模組 (慢/抽象) | Meta-Skill + 反思策略 | 高維、靈活、可遷移 |
| 低層模組 (快/細節) | 任務執行 + 具體觀察 | 低維、專門化 |
| 層級收斂 | Protected Regions + 漸進更新 | 防止 patch 湍流 |
| 1-step gradient | Skill Patch (小改動) | 不追溯完整歷史 |
| Deep Supervision (detach) | 頻繁輕量檢查點 + 截斷 | 不讓舊觀察影響新判斷 |
| ACT 自適應計算 | 按複雜度決定反思深度 | 快想/慢想/深思 |
| PR 維度層級 | 知識抽象度分層 | 高層知識 ≠ 低層知識 |

詳見 `self-improvement-protocol` skill 的「HRM-Enhanced 進化層」章節。
