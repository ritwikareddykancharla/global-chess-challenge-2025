# RLVR Curriculum for Global Chess Challenge 2025

**Objective**: Train an 8B model to achieve <30 ACPL (beating current ~46 ACPL) using Reinforcement Learning with Verifiable Rewards.

**Core Philosophy**: Progressive difficulty. Start by rewarding basic legality, move to material gain, then positional nuances, and finally specialized "trap" avoidance.

---

## Technical Stack
*   **Base Model**: Qwen-2.5-7B-Instruct (SFT-tuned first).
*   **Validation Env**: `chess-env` (from specificed repo).
*   **Oracle**: Stockfish 16 (local binary).
*   **RL Algorithm**: GRPO (Group Relative Policy Optimization) or PPO.
    *   *GRPO Recommended*: Allows generating $N$ thoughts/moves per position and normalizing rewards across the group, heavily rewarding the best thought process.

---

## Phase 1: The "Rules & Formatting" Gate (Epochs 1-2)
*Goal*: Eliminate format errors and illegal moves. The model must learn that "Losing the game" is better than "Crashing the parser".*

*   **Dataset**: 50k Random FENs from Lichess Database (Opening + Middlegame).
*   **Input**: Standard Prompt.
*   **Reward Function**:
    *   `+1.0`: Output matches Valid UCI format AND Move is Legal.
    *   `-1.0`: Invalid format or Illegal move.
*   **Graduation Criteria**: 99.9% Legal Move Rate on validation set.

## Phase 2: Tactical Awareness / "Don't Hang Pieces" (Epochs 3-5)
*Goal*: Teach the model immediate tactical consequences (1-2 ply lookhead).*

*   **Dataset**: 100k "Lichess Puzzle" FENs (Mate in 1, Mate in 2, Hanging Piece).
*   **Reward Function**:
    *   `+1.0`: Move matches Puzzle Solution (Best Move).
    *   `+0.5`: Move is not the best but maintains material equality (CP loss < 50).
    *   `-1.0`: Blunder (CP loss > 200) or Missed Mate.
*   **Verification**: Run against Stockfish Skill Level 0.
*   **Graduation Criteria**: 90% Puzzle Accuracy (Pass/Fail).

## Phase 3: The "ACPL" Grind (Epochs 6-20)
*Goal*: Minimize Average Centipawn Loss on general positions. This is the core leaderboard metric.*

*   **Dataset**: 1M+ High-Elo positions (Both players > 2200).
    *   Mix: 20% Opening, 60% Middlegame, 20% Endgame.
*   **Reward Function (The "Squeeze")**:
    *   Calculate `Movescore` = Stockfish Eval (Depth 20) of Agent Move.
    *   Calculate `Bestscore` = Stockfish Eval (Depth 20) of Best Move.
    *   Let `Diff` = `Bestscore` - `Movescore`.
    *   **Reward** = `e^(-k * Diff)` where `k` is a shaping parameter (e.g., 0.01).
        *   Perfect Move (Diff=0) -> Reward 1.0.
        *   Inaccuracy (Diff=50) -> Reward ~0.6.
        *   Blunder (Diff=200) -> Reward ~0.13.
        *   Illegal -> Reward -0.5.
*   **Implementation Note**: Use `chess-env` to step the board and query Stockfish. Batch processing is critical for speed.

## Phase 4: Adversarial & "Hard" Positions (Epochs 21+)
*Goal*: Robustness against specific weaknesses and "Horizon Effect" problems where LLMs fail.*

*   **Dataset**: "Hard" sets (WAC - Win At Chess, ETP - Extended Tactical Positions).
*   **Opponent Training**:
    *   Instead of static FENs, run **Self-Play** or **vs Stockfish Level 5** games for short episodes (10 moves).
    *   Reward is based on the *final state* of the episode (Win/Loss/Advantage) to encourage planning.
*   **Curriculum Adjustment**: Refocus on positions where the model has historically high ACPL (Active Learning).

---

## Detailed RLVR Loop (GRPO Spec)

For every training step (Batch of $B$ positions):
1.  **Generate**: Sample $G$ completions (e.g., 4) per position.
    *   Format: `<think>...</think><uci_move>...</uci_move>`
2.  **Evaluate**:
    *   Parse `<uci_move>`.
    *   Run Stockfish verification.
    *   Compute Reward $R$ (using Phase 3 formula).
3.  **Advantage**:
    *   For each group of $G$ outputs, normalize rewards: $A_i = \frac{R_i - \text{mean}(R)}{\text{std}(R) + \epsilon}$.
4.  **Update**:
    *   Update policy to increase probability of high-advantage outputs.
    *   *Critical*: The model learns to associate the *Thought Process* (<think>) with the *Result*.

## Hardware Requirement Estimate
*   **Inference (Generation)**: 1 x A100 (80GB) or 2 x A100.
*   **Stockfish Oracle**: CPU bound. Need 32+ vCPUs dedicated to Stockfish to keep up with GPU generation.
