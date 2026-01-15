# Global Chess Challenge 2025 Code Analysis

## Overview
This repository is the starter kit for the Global Chess Challenge 2025. The goal is to build a text-only chess agent that plays legal moves and provides a one-sentence rationale for each move.

## Key Components

### 1. `README.md`
- **Goal**: Build an agent that outputs a thought process and a move in UCI format.
- **Input**: FEN string, side to move, legal moves list.
- **Output**: `<think>reasoning</think> <uci_move>move</uci_move>`.
- **Evaluation**: Round-robin tournaments, ACPL (Average Centipawn Loss) metric compared to Stockfish.
- **Submission**: Hugging Face model + prompt template.

### 2. `local_evaluation.py`
- **Purpose**: Runs local games between your agent (via API) and an opponent (random or Stockfish).
- **Agents**:
    - `OpenAIEndpointAgent`: Connects to an OpenAI-compatible API (e.g., vLLM) to get moves.
    - `StockfishAgent`: Uses a local Stockfish binary.
    - `RandomAgent`: Random moves.
- **Metrics**: 
    - Win/Loss/Draw rates.
    - ACPL (calculated using a local Stockfish analysis after the game).
    - Average time per move.
- **Process**:
    - Uses `chess-env` (submodule) to manage game state and validity.
    - Multithreaded execution for playing multiple games in parallel.
    - Logs games to `logs/` directory as JSON.

### 3. `player_agents/` Directory
- Contains templates and server scripts.
- **`README.md`**: Explains how to create a custom agent using Jinja2 templates.
- **`llm_agent_prompt_template.jinja`**: The standard prompt template.
    - Provides variables: `FEN`, `legal_moves_uci`, `board_utf`, etc.
    - Enforces the output format (reasoning + move).
- **Servers**:
    - `run_vllm.sh`: Script to start a vLLM server (for LLM agents).
    - `stockfish_agent_flask_server.py` & `random_agent_flask_server.py`: Flask servers for rule-based agents (for testing/generating data).

### 4. `chess-env/` Directory
- **Status**: Currently empty on your local machine.
- **Issue**: This is likely a git submodule that hasn't been initialized.
- **Fix**: Run `git submodule update --init --recursive` to fetch the environment code.

## Actionable Next Steps
1. **Initialize Submodules**: Run `git submodule update --init --recursive` to populate `chess-env`.
2. **Setup Environment**: Install dependencies from `requirements.txt`.
3. **Develop Agent**:
    - Choose a base model (e.g., Qwen, Llama).
    - Customize the Jinja2 template if needed.
    - Fine-tune or prompt-engineer the model to output valid moves and reasoning.
4. **Local Testing**:
    - Spin up the model using vLLM (`player_agents/run_vllm.sh`).
    - Run `python local_evaluation.py` to play against Stockfish/Random agents.
5. **Submission**:
    - Push model to Hugging Face.
    - Configure `aicrowd_submit.sh`.
    - Submit via AIcrowd CLI.
