# Inventory Replenishment Agent

A daily agent that decides, for each SKU, whether to **place a purchase order or wait**. It forecasts demand with EWMA, sizes safety stock from forecast error and the target service level, projects inventory over the lead time, and respects minimum order quantities. A day-by-day simulation receives orders after the lead time and compares the agent with a fixed reorder rule.

## Results (76 simulated days, 3 SKUs)

| Policy | Fill rate | Lost units | Stockout days | Holding | Stockout | Ordering | **Total cost** |
|---|---|---|---|---|---|---|---|
| Baseline (fixed s, Q, no forecast) | 91.0% | 501 | 34 | $2,379 | $9,222 | $475 | **$12,076** |
| Agent, naive forecast | 99.6% | 23 | 4 | $4,421 | $470 | $1,300 | **$6,191** |
| **Agent, EWMA (α = 0.3)** | **99.6%** | **25** | **5** | $2,309 | $550 | $1,450 | **$4,309** |

The EWMA agent cuts total cost by **64%** and lost sales by **95%** compared with the baseline, with about the same holding cost. See section 10 of the notebook for the service-level and alpha trade-off analysis.

## How to run

```bash
pip install -r requirements.txt
```

```bash
jupyter nbconvert --to notebook --execute --inplace InventoryReplenishmentAgent.ipynb
```

Or open `InventoryReplenishmentAgent.ipynb` in Jupyter or Google Colab and choose **Run All**. If the `data/` folder is missing, the notebook regenerates the same CSVs (fixed seed).

All settings are in the `AgentConfig` cell: `alpha`, `service_level_override`, `lead_time_override`, `fixed_order_cost`, `max_cover_days`, `warmup_days`.

## Files

| Path | What it is |
|---|---|
| `InventoryReplenishmentAgent.ipynb` | Notebook with code and saved outputs |
| `DESIGN.md` | Design document: agent role, inputs and outputs, guardrails, metrics, ethics and trust |
| `SCALING.md` | Scaling note: cloud deployment, monitoring, reliability, cost controls |
| `data/sales.csv` | Daily sales transactions: `date,sku,qty_sold` (90 days, 3 SKUs) |
| `data/inventory.csv` | Opening stock: `sku,opening_stock` |
| `data/params.csv` | `sku,unit_cost,holding_cost_per_day,stockout_cost,lead_time_days,min_order_qty,service_level` |
| `outputs/agent_daily_log.txt` | Day-by-day agent decisions with rationale |
| `outputs/agent_log.csv` | Full simulation log for all three policies |
| `outputs/kpi_comparison.csv` | KPI table: agent vs. baseline |
| `outputs/service_level_sweep.csv`, `outputs/alpha_sweep.csv` | Trade-off sweeps |
| `outputs/*.png` | Demand, inventory and trade-off charts |
| `outputs/InventoryReplenishmentAgent.html` | Executed notebook as a web page |

## How the agent decides

Each day, for each SKU:

1. Receive any purchase orders due today, then serve demand. Unmet demand is a lost sale.
2. Update the EWMA forecast and its error (rolling RMSE of one-step errors).
3. Safety stock `SS = z(service_level) × σ × √(L + R)`; order-up-to level `S = forecast × (L + R) + SS`.
4. Project stock day by day over `L + R` days (on-hand + arriving POs − forecast).
5. If projected stock at the end is below SS, **ORDER** `max(S − inventory position, min_order_qty)`, capped at 30 days of demand. Otherwise **WAIT**.
6. Log the decision and the numbers behind it.

Example log lines:

```
2025-01-15  day  1  demand  34             end  216
      ORDER | on-hand 216, on-order 0, fc 34.4/d, sigma 6.5, SS 26 | projected stock in 6d = 10 < SS 26 -> order 150 to reach S=233, arrives in 5d; MOQ raised qty from 17 to 150
2025-01-16  day  2  demand  43             end  173
      WAIT  | on-hand 173, on-order 150, fc 37.0/d, sigma 6.7, SS 27 | projected stock in 6d = 101 >= SS 27; ordering now would only add ~$7.50/day holding
```
