# Design Document: Inventory Replenishment Agent
**Agent goal.** For every SKU, keep customers supplied (few stockouts, high fill rate) at the **lowest total cost** (holding + stockout + ordering). The agent runs once per day (review period R = 1 day).

**Inputs (per SKU, per day)**
| Input | Source |
|---|---|
| Demand forecast (units/day) and forecast error (RMSE) | EWMA over `sales.csv`, updated daily |
| On-hand stock, open purchase orders (qty + arrival day) | simulation state, starting from `inventory.csv` |
| Lead time, service level, min order qty, costs | `params.csv` |

**Output.** One decision per SKU per day: `ORDER q units` or `WAIT`, plus a one-line rationale written to the daily log.

**Decision rule.** Look ahead over the *protection period* L + R (the time until a new order could next be replenished). Project stock day by day: on-hand + arriving POs − forecast × days. If the projected stock at the end of the period falls below safety stock, order up to `S = forecast × (L+R) + safety stock`; otherwise wait, because ordering early only adds holding cost.

**Guardrails & business rules**
- **Min order qty:** every PO is at least `min_order_qty`; the log says when MOQ forced a larger order than needed.
- **Lead time:** orders arrive exactly `lead_time_days` later; the projection counts only stock that can physically arrive in the window, and warns when a stockout will happen before a new PO can land.
- **Service level → safety stock:** `SS = z(service_level) × σ_error × √(L+R)`.
- **No over-ordering:** one PO never covers more than `max_cover_days` (30) of forecast demand, and the agent never orders while projected stock is already above safety stock.
- **Holding vs. stockout trade-off:** a higher service level buys fewer stockouts with more stock on hand. Section 10 measures that curve rather than assuming it.

**Metrics**
- **Stockouts:** units of demand not served (lost sales) and the number of SKU-days with any unmet demand.
- **Fill rate:** units served ÷ units demanded.
- **Total cost:** holding (`on_hand × holding_cost_per_day`) + stockout (`lost_units × stockout_cost`) + ordering (fixed cost per PO). Purchase spend (`qty × unit_cost`) is tracked separately as working capital.

**Ethics & trust.** A stockout breaks a promise to a customer, and excess stock ties up the business's cash and can become waste. The agent treats both as real costs instead of maximizing one at the expense of the other. Every decision is logged with the numbers behind it (forecast, safety stock, projection, MOQ effect), so a planner can audit, question or override it. The agent recommends; people keep the final say.
