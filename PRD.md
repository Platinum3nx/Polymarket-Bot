# PRD: Autonomous Polymarket Catalyst Trading Agent (Project "Oracle")

## 1. Product Overview
**Objective:** Build a Python-based autonomous trading agent that capitalizes on *information asymmetry* in prediction markets. The bot will ingest real-time text data (news, announcements), use a Large Language Model (LLM) to calculate the probability of a binary market resolving to "YES", and execute trades on the Polymarket Central Limit Order Book (CLOB) if a profitable mathematical edge exists.
**Language/Tech Stack:** Python 3.10+, `py-clob-client`, Anthropic/Gemini API, Tavily/RSS parsing.

## 2. System Architecture
The system operates on a linear, event-driven loop rather than a continuous high-frequency scanning loop.

### 2.1 Component Breakdown
1.  **The Sensor (Data Ingestion):** Polls specific high-signal sources.
2.  **The Brain (Evaluation):** Parses text and outputs a probabilistic integer.
3.  **The Hands (Execution):** Handles L1/L2 Polymarket authentication, calculates position size, and signs the EIP-712 transaction.

---

## 3. Detailed Component Specifications

### 3.1 Module A: The Sensor (Data Ingestion)
The bot must **not** scan the entire Polymarket platform. It will monitor pre-defined "Watched Markets."
* **Input:** A hardcoded `config.json` containing a list of `market_id` strings and associated search queries (e.g., `market_id: "0x123...", query: "SEC ETF approval news"`).
* **Polling Rate:** Trigger a search via Tavily API or specific RSS feeds every 60 seconds.
* **Filter:** Only pass data to Module B if the text timestamp is within the last 5 minutes.

### 3.2 Module B: The Brain (LLM Evaluation)
This module acts as the probability engine.
* **LLM Model:** Claude 3.5 Sonnet or Gemini 1.5 Pro.
* **Input Prompt Structure:**
    > "You are a prediction market analyst. Market Question: {market_question}. Recent News: {news_text}. Does this news confirm or strongly indicate the outcome of the market? Output strictly a JSON object: `{"probability": float, "confidence": float, "reasoning": "string"}`. The probability must be between 0.00 and 1.00."
* **Validation:** The code must strictly parse the JSON. If the LLM hallucinates formatting, catch the `JSONDecodeError` and return a probability of 0.5 (Neutral/No Action).

### 3.3 Module C: The Hands (Polymarket CLOB Execution)
This is the most critical technical module. The IDE must strictly adhere to Polymarket's `py-clob-client` authentication flow.

* **Authentication (Crucial):** Polymarket requires two-layer authentication.
    1.  **L1 (Private Key):** Read the `PRIVATE_KEY` from environment variables.
    2.  **L2 (API Credentials):** The bot *must* dynamically generate or derive its API keys on startup using the `create_or_derive_api_key()` method.
    3.  **Initialization Code Spec:**
        ```python
        # The IDE must use this exact initialization pattern
        client = ClobClient(
            host="[https://clob.polymarket.com](https://clob.polymarket.com)",
            key=os.getenv("PRIVATE_KEY"),
            chain_id=137, # Polygon Mainnet
            signature_type=0 # 0 for Standard EOA (MetaMask), 1 for Magic/Email
        )
        api_creds = client.create_or_derive_api_key()
        client.set_api_creds(api_creds)
        ```
* **Rate Limits:** The bot must adhere to the CLOB limits (1500 requests / 10s for order books, 3500 requests / 10s for POSTing orders). Since we are event-driven, we will be well under this, but the IDE must include basic backoff/retry logic for HTTP 429 errors.
* **Execution Type:** **Limit Orders Only**. Never use market orders due to low liquidity slippage. Fetch the current Orderbook and place a limit order exactly at the current "Ask" price.

---

## 4. Mathematics & Risk Management (Guardrails)

### 4.1 Position Sizing (The Kelly Criterion)
If the LLM returns a probability (p) that diverges favorably from the current market price (c), the bot must calculate the bet size using the Kelly Criterion adapted for prediction markets:

f* = (p - c) / (1 - c)

*Where:*
* f* = Fraction of bankroll to wager.
* p = LLM's probability (e.g., 0.80).
* c = Current YES share price (e.g., 0.50).

### 4.2 Hardcoded Safety Limits
The IDE must implement the following strict variables. The bot must fail to compile or run if these are missing:
1.  **Fractional Kelly (`KELLY_DIVISOR = 4`):** The raw f* value must be divided by 4 before execution. (e.g., If Kelly suggests betting 40%, the bot bets 10%).
2.  **Absolute Max Trade (`MAX_USDC_PER_TRADE = 25`):** Regardless of the bankroll or Kelly calculation, `min(calculated_bet, MAX_USDC_PER_TRADE)` must be applied.
3.  **Minimum Edge (`MIN_EDGE = 0.10`):** The bot will not trade unless abs(p - c) > 0.10. A 2% edge is not worth the gas/API risk.
4.  **Kill Switch:** Before every trade, fetch the USDC balance. If `current_balance < (starting_balance * 0.70)`, halt all execution and exit the process.

---

## 5. Edge Cases & Error Handling Specs
* **Insufficient Allowance:** The bot must verify the USDC allowance for the Polymarket Exchange Contract. If it is 0, it must log an error and instruct the human operator to approve the contract manually. (Do not let the bot auto-approve infinite funds).
* **Thin Orderbooks:** If the depth of the orderbook at the current Ask price is less than the calculated `bet_size`, the bot must adjust the `bet_size` to match the available liquidity, preventing partial fills from getting stuck.