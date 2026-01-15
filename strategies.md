# Global Chess Challenge 2025: Best Strategies

This document outlines the optimal technical strategies to win the Global Chess Challenge 2025, strictly adhering to the "No External Tools" and "<8B Parameters" constraints.

## 1. Model Architecture & Selection
The strict 8B parameter limit suggests using the strongest available "dense" or "reasoning-focused" 7B execution models.

*   **Primary Recommendation**: **Qwen-2.5-7B-Instruct** (or the base/Math version).
    *   *Why*: Currently the state-of-the-art for reasoning/math/code in the 7B class. Chess notation (algebraic) and board state (FEN) require precise symbolic manipulation, which Qwen excels at.
*   **Alternative**: **Llama-3.1-8B-Instruct**.
    *   *Why*: Strong general reasoning, massive pre-training corpus.
*   **Specialized**: **DeepSeek-Math-7B-RL**.
    *   *Why*: Pre-trained specifically for coherent multi-step reasoning value functions.

## 2. Data Strategy (The "Teacher" Pipeline)
Since the model cannot use search (MCTS/Minimax) at inference time, we must "distill" the search capability into the model's weights via Data-Centric AI.

### A. Dataset Sources
1.  **Lichess Elite Database**: Filter for games where both players are 2400+ Elo.
2.  **CCRL / TCEC Games**: Engine vs Engine games (superhuman quality).
3.  **Puzzle Sets**: Lichess Puzzles (tactics training).

### B. The "Thought" Annotation (Distillation)
The challenge requires a "rationale". We can use this to improve performance via Chain-of-Thought (CoT) distillation.
1.  **Step 1**: Take a high-quality position + Best Move (Stockfish).
2.  **Step 2**: dynamic generation of "Rationales" using a proprietary Teacher Model (e.g., GPT-4o, Claude 3.5 Sonnet, or DeepSeek-V3).
    *   *Prompt for Teacher*: "Explain why Move X is the best move in this FEN position. concise 2-sentence explanation focusing on tactics and strategy."
3.  **Step 3**: Create SFT dataset:
    *   **Input**: `{{FEN}}` ... (matches the official prompt template)
    *   **Output**: `<think>Teacher_Rationale</think> <uci_move>Stockfish_Best_Move</uci_move>`

## 3. Training Pipeline

### Phase 1: Supervised Fine-Tuning (SFT)
*   **Objective**: Teach the model the *format*, *legality*, and *basic strategic patterns*.
*   **Data**: 1M+ samples of (FEN -> Rationale + Move).
*   **Loss**: Standard Next-Token Prediction on the Output.
*   **Evaluation**: Check % Legal Moves and ACPL on a holdout set.

### Phase 2: Reinforcement Learning with Verifiable Rewards (RLVR)
This is the winning differentiator. Since we have a "Ground Truth" verifier (Stockfish + Rules) that is cheap to run *during training*, we can use PPO or GRPO to exceed the specific SFT limitations.

*   **Algorithm**: **PPO** (Proximal Policy Optimization) or **GRPO** (Group Relative Policy Optimization - simpler, no value network needed).
*   **Environment**: `chess-env` (provided in kit).
*   **Reward Function ($R$)**:
    *   $R_{legal}$: $+1$ if move is legal, $-1$ (and end episode) if illegal.
    *   $R_{quality}$: Scale based on Centipawn Loss (CPL) relative to Stockfish.
        *   $R_{cp} = \max(0, 1 - \frac{CPL}{100})$ (Example: perfect move = +1, blunder = 0).
    *   $R_{outcome}$: $+10$ for Checkmate, $-10$ for being Checkmated.
*   **Training Loop**:
    1.  Model generates output for a batch of FENs.
    2.  Parser extracts `<uci_move>`.
    3.  `chess-env` validates legality and calculates rewards via Stockfish.
    4.  Update model weights to maximize Reward.

## 4. Constraint Optimization (Inference)
*   **Prompt Engineering**: Even though the template is fixed, your model effectively learns to "fill in the blanks".
*   **"Thinking" Space**: The prompt limits reasoning to "2 sentences".
    *   *Hack*: Train the model to output extremely *dense* reasoning (Chess algebraic notation variants) within those 2 sentences to pack more "search" into the context window.
    *   *Example*: "Threat:Re8. Plan:Nd5->f6. Best:Nd5." (This counts as sentences but is actually a search path).

## 5. Execution Plan (Step-by-Step)
1.  **Setup**: Deploy `vLLM` server locally with Qwen-2.5-7B to test the `local_evaluation.py` loop.
2.  **Baseline**: Run the starter kit `RandomAgent` and `StockfishAgent` to establish baseline ACPL on your hardware.
3.  **Data Prep**: Download 100k Lichess Elite games. Parse to FEN + Move.
4.  **SFT**: Fine-tune Qwen-2.5-7B LoRA on this dataset.
5.  **Evaluate**: Run `local_evaluation.py` on the SFT model.
6.  **RLVR**: Implement a PPO loop using the SFT model as the reference policy. Train against Stockfish Level 0, then Level 5, then Level 20.

## 6. Resources & Tools
*   **Engine**: Stockfish 16 (AVX2 build for speed).
*   **Library**: `python-chess` (already in `chess-env`).
*   **Training**: `unsloth` (for 2x faster training on single GPU) or `TRL` (HuggingFace Transformer Reinforcement Learning).
