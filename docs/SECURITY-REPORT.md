# SECURITY REPORT

Date: 2026-03-29

Scope:
- `python src/main.py`
- `python src/backtester.py`

Method:
- Static review of the two CLI entrypoints and the modules they can reach during normal execution.
- Focus areas: CLI inputs, environment variables, subprocess usage, outbound network calls, LLM prompt/data flow, secret handling, filesystem impact, and denial-of-service risk.
- This report does not claim exploitability from an untrusted local user unless noted. Several issues assume a hostile `.env`, hostile upstream data, or a user tricked into selecting risky options.

## Executive Summary

The most serious risk in these startup paths is the `--ollama` branch. It can execute a remote install script via `curl ... | sh`, and it trusts environment-controlled Ollama endpoints for follow-up API operations. Outside that branch, the biggest practical risks are availability and spend abuse: the code accepts unbounded ticker/date inputs, has long retry sleeps, lacks network timeouts for the financial data provider, and keeps an unbounded in-memory cache.

There is no obvious arbitrary code execution path from LLM output alone in the reviewed flow. The portfolio decision path does apply schema validation and action coercion before executing trades. That said, external news/provider data still enters prompts with minimal sanitization, so data poisoning and prompt injection remain meaningful decision-integrity risks.

## Findings

### 1. High: Remote code execution by design in Ollama auto-install flow

Severity: High

Affected paths:
- `src/cli/input.py:119-146`
- `src/utils/ollama.py:114-204`
- `src/utils/ollama.py:152`
- `src/utils/ollama.py:168`

Details:
- When a user runs either startup path with `--ollama`, `parse_cli_inputs()` calls `ensure_ollama_and_model()`.
- If Ollama is not installed, `install_ollama()` may execute:
  - `bash -c "curl -fsSL https://ollama.com/install.sh | sh"`
- This is a direct remote-script execution pattern. It gives full trust to a live network response at runtime.

Why this matters:
- A compromised download source, TLS interception, hostile mirror, or supply-chain incident would become arbitrary code execution on the local machine.
- Because this sits on a normal interactive startup path, it is easy for users to trigger without understanding the trust boundary.

Recommendation:
- Remove automatic `curl | sh` execution entirely.
- Replace it with a manual-install instruction flow only.
- If automation must remain, download a pinned artifact, verify checksum/signature, and require an explicit second confirmation before execution.
- Clearly label this as a privileged local install action in the UI.

### 2. High: Environment-controlled outbound endpoint trust enables SSRF-like and secret-redirection risks

Severity: High

Affected paths:
- `src/main.py:25`
- `src/llm/models.py:147-153`
- `src/llm/models.py:175-180`
- `src/utils/ollama.py:17-30`
- `src/utils/ollama.py:311-318`
- `src/utils/docker.py:8-30`
- `src/utils/docker.py:33-124`

Details:
- `.env` is loaded automatically on startup.
- `OPENAI_API_BASE` is passed directly into `ChatOpenAI(...)`.
- `OLLAMA_BASE_URL` is accepted from the environment and used to build arbitrary request targets.
- If `OLLAMA_BASE_URL` is set, the code delegates to Docker helper functions that perform `GET`, `POST`, and `DELETE` requests against that URL.

Why this matters:
- Anyone who can alter the environment or `.env` can silently redirect model traffic and management operations to an attacker-controlled service or an internal host.
- In the OpenAI-compatible path, API keys may then be sent to the attacker-controlled endpoint.
- In the Ollama path, the app can be tricked into probing and mutating arbitrary reachable HTTP services.

Recommendation:
- Treat `OPENAI_API_BASE` and `OLLAMA_BASE_URL` as privileged configuration.
- Enforce an allowlist of approved hosts or require explicit `--allow-custom-endpoint`.
- Reject non-HTTP(S) schemes and loopback/LAN targets unless the user explicitly opts in.
- Log the resolved endpoint once at startup with a strong warning when it is non-default.

### 3. Medium: Financial API requests have no timeout and can hang indefinitely

Severity: Medium

Affected paths:
- `src/tools/api.py:29-60`
- `src/tools/api.py:48-50`
- `src/backtesting/engine.py:81-97`
- `src/backtesting/engine.py:114-141`

Details:
- `_make_api_request()` calls `requests.get()` and `requests.post()` without a timeout.
- On top of that, 429 responses trigger sleeps of 60s, 90s, and 120s.
- `BacktestEngine` eagerly prefetches multiple datasets per ticker before the run begins.

Why this matters:
- A slow or blackholed upstream can stall both CLI entrypoints indefinitely.
- An attacker controlling a proxy, DNS, or a trusted-but-unhealthy upstream can cause long-running hangs with minimal effort.
- This is especially damaging in `src/backtester.py`, where the fan-out is much larger.

Recommendation:
- Add bounded connect/read timeouts to every provider request.
- Replace long blocking sleeps with capped exponential backoff and a total request budget.
- Surface a clear failure state instead of hanging.

### 4. Medium: Unbounded input fan-out can cause cost and resource exhaustion

Severity: Medium

Affected paths:
- `src/cli/input.py:23-43`
- `src/cli/input.py:67-70`
- `src/cli/input.py:190-209`
- `src/backtester.py:43-66`
- `src/backtesting/engine.py:81-189`
- `src/main.py:46-89`

Details:
- There is no limit on ticker count.
- There is no upper bound on backtest date span.
- The backtester prefetches prices, metrics, insider trades, and news for every ticker, then invokes the full agent workflow for every business day in the range.
- The main path can also run many analysts at once, each of which may trigger more provider and LLM calls.

Why this matters:
- A single command can explode into a very large number of external requests and model invocations.
- This is a practical denial-of-service and bill-shock vector, especially on shared machines or CI runners.
- Because this is CLI-driven, the threat model is usually accidental misuse or social engineering rather than remote exploitation.

Recommendation:
- Cap ticker count and backtest duration by default.
- Add a `--force-large-run` override with an estimated request/cost summary.
- Add concurrency and total-call budgets.
- Abort early when projected fan-out crosses a threshold.

### 5. Medium: Unbounded in-memory cache can be abused for memory exhaustion

Severity: Medium

Affected paths:
- `src/data/cache.py:1-71`
- `src/tools/api.py:63-97`
- `src/tools/api.py:100-140`
- `src/tools/api.py:186-250`
- `src/tools/api.py:253-317`

Details:
- The cache is process-global, in-memory, has no TTL, no eviction policy, and no size limits.
- Data is merged and retained across requests for prices, metrics, insider trades, and news.
- The backtest path is particularly likely to accumulate large result sets.

Why this matters:
- Large ticker lists or long runs can grow process memory without bound.
- In a long-lived service or repeated interactive session, this can degrade or crash the process.

Recommendation:
- Add TTL and size/entry limits.
- Bound cached result sizes per ticker/query type.
- Prefer keyed exact-match caching with eviction over unbounded merge-only growth.

### 6. Medium: External data is injected into LLM prompts with minimal hardening

Severity: Medium

Affected paths:
- `src/agents/news_sentiment.py:76-85`
- `src/agents/portfolio_manager.py:212-257`
- `src/utils/llm.py:10-84`

Details:
- News headlines are inserted directly into the prompt used for sentiment classification.
- Other agent paths also package provider data into prompt content and rely on the LLM to return structured decisions.
- There is no content provenance marker, prompt delimiter hardening, or adversarial text stripping.

Why this matters:
- A hostile news title or poisoned upstream payload can steer the model toward incorrect classifications or malformed reasoning.
- This is not host compromise, but it is a real decision-integrity issue in an automated trading workflow.

Recommendation:
- Wrap untrusted data in explicit quoted blocks and label it as untrusted.
- Add prompt rules that instruct the model not to follow instructions contained in market/news content.
- Consider lightweight sanitization for obvious instruction-like patterns.
- Log prompt-origin metadata for incident review.

### 7. Low to Medium: Verbose console output can leak sensitive operational context

Severity: Low to Medium

Affected paths:
- `src/main.py:30-42`
- `src/graph/state.py:18-43`
- `src/utils/display.py:17-240`
- `src/agents/news_sentiment.py:144-152`
- `src/agents/risk_manager.py:205-214`

Details:
- Parse failures in `parse_hedge_fund_response()` print the raw model response.
- `show_reasoning` and display helpers print detailed agent reasoning, including trading logic and portfolio-related context.
- Several agents send reasoning blobs into progress/output functions.

Why this matters:
- In multi-user terminals, shell history capture, CI logs, or shared observability sinks, this can expose proprietary strategy reasoning and run-specific portfolio state.
- This is more of a confidentiality/control issue than a direct exploit.

Recommendation:
- Add a quiet/safe logging mode and make it the default.
- Redact or truncate raw model responses on parse failure.
- Gate detailed reasoning behind an explicit debug flag with a warning.

### 8. Low: Invalid analyst names can crash startup

Severity: Low

Affected paths:
- `src/cli/input.py:77-78`
- `src/main.py:100-130`
- `src/main.py:112-124`

Details:
- `--analysts` accepts arbitrary comma-separated strings.
- `create_workflow()` indexes directly into `analyst_nodes[analyst_key]` without validating membership first.

Why this matters:
- A malformed CLI input can terminate the run before work starts.
- This is primarily an availability/robustness issue, but it is still a useful hardening fix.

Recommendation:
- Validate all analyst names against the configured set before workflow creation.
- Fail with a clean error message listing allowed values.

## Positive Controls Observed

These reduce risk in the reviewed paths:

- Portfolio actions are schema-constrained with `Literal["buy", "sell", "short", "cover", "hold"]` in `src/agents/portfolio_manager.py:13-21`.
- Backtest execution re-validates actions and coerces unknown values to `hold` in `src/backtesting/controller.py:35-59`.
- Trade execution ignores non-positive quantities in `src/backtesting/trader.py:15-31`.
- The reviewed paths do not use `eval`, `exec`, `pickle.loads`, or shell interpolation based on ticker/news/model output.

## Prioritized Remediation Plan

1. Remove automatic `curl | sh` installation from the Ollama startup path.
2. Add allowlists and explicit opt-in for custom `OPENAI_API_BASE` and `OLLAMA_BASE_URL`.
3. Add request timeouts and a global retry budget for `src/tools/api.py`.
4. Add hard limits for ticker count, analyst count, and backtest date span.
5. Add cache eviction, TTL, and memory guards.
6. Harden prompts against instruction-bearing news/provider content.
7. Reduce default console verbosity and redact raw model output on failures.
8. Validate analyst names before workflow construction.

## Bottom Line

If these scripts are used locally by a trusted developer with a trusted `.env`, the main near-term risk is accidental cost/resource blow-up. If the environment or upstream endpoints can be influenced, the `--ollama` path becomes the dominant security concern because it combines remote installation, subprocess execution, and environment-directed network trust.
