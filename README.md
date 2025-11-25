# Agentic Quant Research Lab

### Problem Statement

Quantitative analysts and retail traders lose valuable opportunities because comprehensive market research is slow, disjointed, and prone to human error.

The core problems:
- **Data Overload:** Bridging the gap between hard numerical data (price action, volatility) and soft qualitative data (news sentiment) requires switching between multiple disjointed tools.
- **Manual Labor:** Fetching OHLCV data, calculating technical indicators, and reading hundreds of headlines for every ticker is an unscalable drain on time.
- **Missing Signals:** Humans often miss sophisticated "divergence" signals—such as a stock price rising despite negative news ("climbing a wall of worry")—because they cannot synthesize multimodal data fast enough.
- **Backtesting Bottlenecks:** Validating a trade idea requires coding a backtest from scratch every time, which discourages rigorous testing.

The Agentic Quant Research Lab is designed to automate the "grunt work" of research: it autonomously ingests data, engineers features, detects signals, and validates them with math, allowing the human trader to focus on high-level strategy.

![agentic-quant-lab](thumbnail.png)

---

### Why agents are the right solution

This problem is naturally an assembly line of expert tasks:
- **ETL:** Extracting and cleaning real-time market data and news.
- **Analysis:** Synthesizing conflicting data points (e.g., "Technicals are good, Sentiment is bad").
- **Reporting:** Drafting a professional, evidence-based research note.

Agents fit because:

**Specialization:**
- A **Data Agent** is optimized for reliable API interaction and state management.
- A **Signal Agent** is optimized for pattern recognition and hypothesis generation.
- A **Reporting Agent** is optimized for mathematical validation and clear communication.

**Workflow & Safety:**
- **Sequential workflow:** Using a `SequentialAgent` guarantees data is fetched and cleaned *before* analysis begins, preventing hallucinations based on missing data.
- **Tool use:** Agents reliably call tools like `fetch_market_data` and `local_backtest` instead of trying to do math or quote prices from their training data.
- **Shared State:** Instead of passing massive dataframes through the LLM context window (which is expensive and error-prone), agents read/write to a shared `ToolContext` state.
- **Safety:** A custom guardrail plugin ensures the agent analyzes data but never gives reckless financial advice (e.g., "Go all in"), protecting the user and the platform.

---

### What I created — overall architecture

**Project Name:** Agentic Quant Research Lab

**High-level architecture:** Sequential Multi-Agent System with Shared State

#### Input
User prompt: "Run a research cycle on NVDA."

#### Agent A – “The Data Engineer”
- **Type:** Agent (`Quant_Data_Agent`)

**Responsibilities:**
- Call `fetch_market_data(ticker)` to retrieve historical pricing via `yfinance` and save it to the shared state.
- Call `fetch_news_sentiment(ticker)` via `duckduckgo_search` to get real-time headlines (with a robust fallback mechanism if the API is blocked).
- Call `compute_simple_factors` to calculate Momentum and Volatility metrics.
- **Output:** A JSON summary of available data (not the raw data itself).

#### Memory Service – “The Shared State”
- **Implementation:** `ToolContext` state dictionary + `InMemoryMemoryService`.

**Stores:**
- Heavy DataFrames (OHLCV data).
- Raw news lists.
- Computed factor arrays.

Accessed via tools to prevent context window bloat.

#### Agent B – “The Signal Analyst”
- **Type:** Agent (`Quant_Signal_Research_Agent`)

**Responsibilities:**
- Read the data summary from Agent A.
- Analyze the sentiment of the fetched headlines.
- Cross-reference sentiment with technical factors.
- **Output:** A structured `signal_spec` containing a hypothesis (e.g., "Bullish Divergence detected").

#### Agent C – “The Reporter”
- **Type:** Agent (`Quant_Reporting_Agent`)

**Responsibilities:**
- Read the `signal_spec`.
- Call `local_backtest` to mathematically validate the hypothesis against the historical data stored in state.
- Draft a formal Research Note:
  - Cites specific metrics (Sharpe Ratio, CAGR).
  - Summarizes news sentiment.
  - Provides a clear conclusion.
- **Output:** Final Markdown report.

#### Orchestrator – “AgenticQuantLab_Workflow”
- **Type:** `SequentialAgent`
- `sub_agents = [data_agent, signal_agent, report_agent]`

**Enforces the pipeline:**
Ingest Data → Detect Signal → Validate & Report.

#### Safety Plugin – “NoTradeAdviceGuardrail”
- **Type:** `BasePlugin`
- **Hook:** `before_agent_callback`

**Behavior:**
- Checks user input for high-risk financial keywords (e.g., "buy", "sell", "short", "go all in").
- If triggered, logs a warning and can block the request to ensure Responsible AI compliance.

---

### Demo — how the solution works from a user’s perspective

#### Request step
The user types: "Run a research cycle on NVDA (Nvidia). Find a signal and write a report."

#### Data phase
The Data Agent:
- Connects to Yahoo Finance to pull the last 252 days of price data.
- Scrapes DuckDuckGo for the latest news headlines.
- Computes technical factors (e.g., 5-day Momentum).
- Saves all heavy data to the session state hidden from the LLM's context window.

#### Analysis phase
The Signal Agent:
- Notices that while the news is negative (e.g., "Competition fears"), the price momentum is positive.
- Formulates a "Climbing a Wall of Worry" hypothesis: The stock is resilient despite bad news.

#### Reporting phase
The Reporting Agent:
- Runs a vector backtest on the chosen momentum factor.
- Determines the strategy yielded a **Sharpe Ratio of 1.89**.
- Generates the final note:
> "**Conclusion:** The 5-day momentum factor demonstrates a positive historical performance with a favorable Sharpe Ratio. The absence of price drops despite negative news sentiment suggests strong underlying demand."

#### Safety checks
If the user asks, "Should I bet my life savings on this?", the `NoTradeAdviceGuardrail` intercepts the request and prevents the agent from responding.

---

### The Build — how I created it, tools & technologies used

#### Core stack

##### Agent framework
Google Agent Development Kit (ADK):
- `SequentialAgent` for deterministic orchestration.
- `ToolContext` for managing shared state between agents.
- `BasePlugin` for safety guardrails.
- `LoggingPlugin` for observability and debugging.

##### Model
Gemini 2.5 Flash (via `Gemini(model="gemini-2.5-flash")`):
- Chosen for high throughput and low latency, essential for multi-step reasoning chains.

##### Custom tools (Function Tool pattern)
- `fetch_market_data`
  - Integrates `yfinance` to pull real-world OHLCV data.
  - Saves data to `context.state` to avoid token limits.
- `fetch_news_sentiment`
  - Integrates `duckduckgo_search`.
  - **Innovation:** Includes a "Circuit Breaker" fallback. If the live search is rate-limited, it returns cached/fallback data so the pipeline doesn't crash.
- `local_backtest`
  - Uses `numpy` and `pandas` to perform vector-based simulation of trading strategies.

#### Agents
- `data_agent`: ETL specialist.
- `signal_agent`: Qualitative/Quantitative synthesis expert.
- `report_agent`: Validation and communication expert.

#### Orchestration
- `quant_workflow = SequentialAgent(...)`
- Ensures that analysis never happens before data is fully prepped and cleaned.

---

### If I had more time, this is what I’d do

If I extend the Agentic Quant Research Lab beyond this blueprint, I’d:

- **Integrate a Vector Database for "Market Memory"**
  - Swap `InMemoryMemoryService` for `Vertex AI Vector Search`.
  - Allow the agent to recall historical market regimes (e.g., "This setup looks like the 2020 crash") to provide deeper context.

- **Add a "Human-in-the-Loop" Approval Step**
  - Pause the workflow after the Signal Agent proposes a hypothesis.
  - Present the hypothesis to the human trader.
  - Only proceed to the expensive Backtest phase if the human clicks "Approve."

- **Expand Tooling to Fundamentals**
  - Add a tool to fetch SEC filings (10-K, 10-Q) via EDGAR.
  - Allow the agent to analyze balance sheet health alongside price action.

- **Live Paper Trading**
  - Connect the final output to a paper-trading API (like Alpaca).
  - Allow the agent to execute the trade in a sandbox environment if the backtest metrics exceed a certain threshold.

That roadmap would turn the Agentic Quant Research Lab from a powerful analysis tool into a fully autonomous algorithmic trading assistant.
