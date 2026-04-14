# Z.ai Coding Plan Integration Guide

This repo already has a **plan/design/code/backtest artifact pipeline**. To integrate with a Z.ai-style coding plan, the fastest path is to map Z.ai plan payloads into the existing `planner_output.json` contract and then let current UI/services consume it.

## 1) What already exists (good integration points)

- Run artifacts persist under per-run folders with key files like `req.json`, `planner_output.json`, `design_spec.json`, and `state.json`.
- `RunStateStore.save_planner_output(...)` already writes planner payloads to `planner_output.json`.
- The UI service (`load_run_context`, `infer_indicator_periods`, `_compute_fetch_start_date`) already reads `planner_output.json` and expects fields under `coding_contract` and `requirements.context`.
- API responses (`RunResponse`) expose `planner_output` directly.

**Implication:** if Z.ai plan output can be transformed into this shape, most of the backend/UI behavior works immediately without major refactoring.

## 2) Minimal target schema for compatibility

For best compatibility with existing code, produce/transform to:

```json
{
  "requirements": {
    "context": {
      "codes": ["AAPL", "MSFT"],
      "start_date": "2024-01-01",
      "end_date": "2024-12-31"
    }
  },
  "coding_contract": {
    "target_scope": ["AAPL", "MSFT"],
    "start_date": "2024-01-01",
    "end_date": "2024-12-31",
    "data_lookback_days": 60,
    "input_logic": {
      "parameters": {
        "signal_params": {
          "fast_ma": 10,
          "slow_ma": 30
        }
      }
    },
    "data_requirements": [
      {"symbol_scope": "AAPL,MSFT"}
    ]
  }
}
```

This is enough for:
- run-context reconstruction,
- indicator period inference,
- lookback-aware market data rebuilding in run analysis.

## 3) Integration architecture (recommended)

### A. Add a Z.ai plan adapter module
Create e.g. `agent/src/integrations/zai_plan_adapter.py`:

- `normalize_zai_plan(raw: dict) -> dict`
- map/rename Z.ai fields to the internal schema above,
- date normalization to `YYYY-MM-DD`,
- fallback logic for symbols/codes.

### B. Ingestion path options

1. **API-first integration (recommended):**
   - Add endpoint `POST /v1/runs/{run_id}/planner/zai`.
   - Validate payload, normalize via adapter, persist via `RunStateStore.save_planner_output`.

2. **Runtime tool integration:**
   - Add a dedicated internal tool (e.g., `plan_zai`) that calls adapter and returns normalized planner payload.
   - Keep persistence through existing `persist_tool_result("plan", ...)` behavior.

### C. Observability

- Save both:
  - normalized payload to `planner_output.json` (existing behavior),
  - raw Z.ai payload to `zai_planner_raw.json` for debugging/audit.
- Include `source: "zai"` + `adapter_version` in normalized metadata.

## 4) Field mapping checklist (Z.ai -> internal)

- `goal` / `user_goal` -> optional top-level passthrough (already used by run listing fallback logic).
- instruments list -> `coding_contract.target_scope` and `requirements.context.codes`.
- date range -> `coding_contract.start_date/end_date` and `requirements.context.start_date/end_date`.
- warmup/lookback -> `coding_contract.data_lookback_days`.
- strategy params -> `coding_contract.input_logic.parameters.signal_params`.
- data scope list -> `coding_contract.data_requirements[].symbol_scope`.

## 5) Validation rules (important)

- Hard-fail if no symbols after normalization.
- Soft-fallback date handling:
  - if absent, allow existing request context to provide dates,
  - always normalize to `YYYY-MM-DD` before persist.
- Clamp `data_lookback_days` to a safe range (e.g., `0..3650`).
- Preserve unknown Z.ai fields under `metadata.zai_extra` for forward compatibility.

## 6) Rollout plan

1. Ship adapter + endpoint behind env flag `ENABLE_ZAI_PLAN_INGEST=1`.
2. Add unit tests for mapping/normalization edge cases.
3. Add one API integration test ensuring `planner_output` appears in run response.
4. Enable in staging and verify run detail UI renders dates/codes/indicators correctly.
5. Gradually enable in production.

## 7) Why this approach fits this codebase

- It leverages existing planner persistence and UI behavior rather than introducing a second planning format.
- It keeps Z.ai coupling localized to one adapter boundary.
- It is reversible and low-risk: turning off ingestion does not affect current run flow.

