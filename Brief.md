# GmLive: LLM-Driven GamesMaster Brief

## Purpose of the Project
GmLive aims to create an AI-powered GamesMaster (GM) that runs locally to orchestrate Tabletop Role-Playing Games (TTRPGs). The system will facilitate real-time interactions with players by listening to their conversations, understanding the context and actions, and responding organically as a human GM would. It will manage the game state, orchestrate scenes, handle player decisions and consequences, and vocally deliver its responses, creating an immersive, hands-free TTRPG experience.

## Functional Details
In a typical TTRPG setting, a group of players sit around a table, conversing both in and out of character. GmLive will integrate into this flow seamlessly:

*   **Real-time Listening, VAD, & Diarization**: The system will continuously capture audio from the table. To prevent transcription hallucinations on open mics, Voice Activity Detection (VAD) will gate the audio before it hits the transcriber. The system will also handle Acoustic Echo Cancellation (AEC) (or require push-to-talk) so the GM does not transcribe its own voice. It will then transcribe the speech to text, identifying which player is speaking.
*   **Context, Action Processing, & State Management**: The core system will maintain the campaign setting, NPC knowledge, and the discrete history of the current session. It must differentiate between out-of-character banter, direct questions to the GM, and definitive character actions using strict deterministic state management for game mechanics.
*   **Turn/Scene Management**: While GmLive listens passively to build context, players will rely on a physical or digital button to "hand over" the turn to the GM to resolve a scene or describe the outcome of an action. (Dynamic "GM Interrupts" based on real-time text analysis are out of scope for the MVP to preserve GPU resources).
*   **Voice Synthesis (TTS)**: The GM's output will be spoken aloud using a high-quality text-to-speech engine. To maintain immersion and reduce Time-To-First-Audio (TTFA), the LLM's output must be streamed to the TTS engine chunk-by-chunk (e.g., sentence-by-sentence).

## Architecture: Universal Ledger Framework (ULF)
To achieve true flexibility across any TTRPG system (D&D 5e, Call of Cthulhu, Cyberpunk, etc.), GmLive uses a mechanic-agnostic orchestration layer called the Universal Ledger Framework (ULF).

The ULF treats TTRPG rules as a "Software Overlay" on top of a generic "World Engine". It operates in a three-layer stack:

1.  **The Engine (Python)**: Handles the "Physics" of the game (Entity Registry, Graph Connectivity, Temporal Decay, Perception Gating). It does not know what a "Saving Throw" or "Spell Slot" is; it only tracks generic Quantities, Timers, and Logic Gates.
2.  **The Rule-Schema (JSON/Markdown)**: A configuration document (e.g., `rules_5e.json`) that defines the specific math, vocabulary, and logic of a game (e.g., "In this game, a 'Hard' task is a DC 20, and the status effect is 'Blinded'").
3.  **The Arbitrator (LLM)**: Ingests the Rule-Schema and uses the Engine's generic tools to enforce those specific rules, acting as the bridge between player intent and the backend physics.

Instead of game-specific function tools (like `update_hp`), the LLM calls **Universal Primitive Operations**:
*   `register_entity`: Creates an object/actor using a generic metadata blob.
*   `register_temporal_state`: Tracks effects with a duration (e.g., Rounds, Hours).
*   `resolve_gate`: Returns a target number based on the schema (e.g., passing "Hard" returns 20 for D&D, or 1/2 skill value for CoC).
*   `check_passive_trigger`: Checks if a player's location intersects a hidden entity based on their generic tags.
*   `sync_world_clock`: Decrements active integers/timers in the state registry regardless of temporal units.
*   `query_rules_engine`: Searches both the rules vector DB and Knowledge Graph for mechanical interactions.
*   `query_adventure_lore`: Searches the adventure vector DB for room descriptions, NPC backgrounds, and custom mechanics.

## Coding and Engineering Standards
*   **Language**: Exclusively Python.
*   **Environment**: A Python `venv` will be used to isolate dependencies.
*   **Architecture**: Strict multi-process, decoupled design from Day 1. The three main components (Speech-to-Text, LLM GM module, Text-to-Speech) will run as separate executables to bypass Python's Global Interpreter Lock (GIL) and ensure non-blocking real-time audio processing. Robust Inter-Process Communication (IPC) via ZeroMQ or Redis is required.
*   **VRAM Lifecycle Management**: Given the high memory footprint of concurrent STT, LLM, and TTS models, the architecture must include a strategy for dynamically loading/unloading models between VRAM and System RAM, or highly restricting model sizes/quantization, to prevent Out-Of-Memory (OOM) errors on consumer GPUs.
*   **Hardware Acceleration**: An Nvidia GPU (CUDA) will be utilized for intensive machine learning workloads to ensure the lowest latency possible.
*   **Code Quality**: `ruff` will be used for blazingly fast formatting and linting. Type hinting (`mypy`) is highly encouraged.
*   **Testing Frameworks**: `pytest` will be the exclusive framework used for unit and mocked functional testing. Integration suites (hitting live LLMs) are strictly decoupled into `integration_tests/` and executed via standalone scripts to prevent pipeline deadlocks on CI runners. `pytest-mock` or `unittest.mock` will be heavily utilized to simulate LLM responses, audio input streams, and TTS outputs, allowing testing without live ML models spinning up. Ensure cleanup scripts are hooked into Pytest's `conftest.py` to prevent hanging ZMQ ports.
*   **Testing Adherence**: A strong culture of testing must be maintained. Core domain logic (the GM orchestration, memory management) must achieve high unit test coverage (target: >80%). Functional tests will simulate complete user interactions (simulated game events -> generated responses). All code must pass linting and unit tests before merging.
*   **UI Engineering (React Dashboard)**: The front-end dashboard is built using Vite, React (TypeScript), and strictly relies on standard UI best practices. ESLint is enforced. 
*   **UI Aesthetics**: Vanilla CSS is used exclusively for styling (No Tailwind). Designs must prioritize a premium visual language (e.g. glassmorphism, custom typography, glowing accents, and dark modes) while ensuring raw JSON is **never** presented to the user. Instead, data must be structured into neatly labeled tables and fields.
*   **Documentation**: Comprehensive docstrings for all functions/classes and a clear README for setup instructions, considering the complexity of local ML environments.

## Technical Details (Modules & Libraries)
*   **Speech-to-Text (STT) & Diarization**:
    *   *Silero VAD* for lightweight Voice Activity Detection to chunk audio before passing it to the transcriber.
    *   *NVIDIA Parakeet-TDT-1.1b* via *nemo_toolkit* for world-class, low-latency, GPU-accelerated transcription.
    *   *PyAnnote.audio* for speaker diarization.
        *   **Current (MVP)**: Basic chunk-level diarization using the `pyannote/speaker-diarization-community-1` pipeline.
        *   *Technical Fixes*: Implemented audio flattening to prevent tensor shape mismatches (the "Matrix Shape Bug").
        *   *Optimization*: Supports `min_speakers` and `max_speakers` hints via environment configuration to improve clustering accuracy in short audio segments.
        *   **Long-term**: "Voice Enrollment" Module. Establish distinct profiles (Voice Fingerprints/Embeddings) upfront where each player reads a short script. Real-time chunks will use cosine similarity against the profile database for cross-session identification. (Fallback/Feedback idea: Continuous Rolling History clustering where the LLM assigns names based on roleplay).
*   **LLM Backend**:
    *   Local model execution served via *LM Studio* or *Ollama*. This allows us to rapidly swap between capable models (e.g., Llama 3 8B at 4-bit) via a standard OpenAI-compatible REST API.
*   **LLM Orchestration (Langflow)**:
    *   GmLive has pivoted to **Langflow** as its primary multi-agent orchestration engine. This allows for rapid visual prototyping of RAG chains, intent routing, and agentic behaviors (The Director, The Rules Lawyer, The Skald) without hardcoding logic in Python.
    *   The Python backend remains the "System Host," managing hardware (STT/TTS), local memory persistence, and ZMQ-based IPC.
*   **Hybrid Knowledge Engine (RAG)**:
    *   *Knowledge Graph (KG)*: Uses `networkx` to map relational logic and mechanical dependencies (Rulebook "Physics").
    *   *Metadata-Augmented Vector RAG*: Uses `ChromaDB` for semantic search of flavor text and lore (Adventure "Content").
    *   *Agentic Orchestrator*: The GM acts as a router, sequentially querying these engines to resolve player actions.
*   **Text-to-Speech (TTS)**:
    *   Fast, local, GPU-accelerated TTS libraries like *Piper TTS* or *Kokoro TTS* to generate natural-sounding synthetic voice output. Must support chunked generation (streaming sentence-by-sentence).
*   **Audio Routing & Hardware Input**:
    *   *PyAudio* or *SoundDevice* for capturing microphone input and playing back generated audio. Must incorporate Acoustic Echo Cancellation (AEC).
    *   *Keyboard* or *pynput* to capture the global hotkey/button press that signals the GM to respond.
