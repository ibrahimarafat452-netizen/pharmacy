# Contract: Recommendation Service

**Module**: AI Recommendation Engine (Phase 9)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IRecommendationService

Calculates and returns inventory recommendations. Permission: Pharmacist+ for viewing, Admin for threshold configuration.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CalculateAll | — | void | Runs all recommendation types for all active products. Replaces old recommendations. Can be triggered manually or on schedule |
| CalculateForProduct | productId | List\<Recommendation\> | All recommendation types for one product |
| GetRecommendations | type?, priority?, pagination | PagedList\<Recommendation\> | Filterable by type and priority |
| GetProductRecommendations | productId | List\<Recommendation\> | All recommendations for one product |
| GetCriticalRecommendations | — | List\<Recommendation\> | Priority = Critical, sorted by score DESC |

---

## Recommendation Calculator Contracts

Each calculator is an independent class implementing a common interface.

### IRecommendationCalculator

| Method | Input | Output |
|--------|-------|--------|
| Calculate | productId, salesHistory, stockData, config | Recommendation? |

### Calculator Types

| Calculator | Inputs | Score Logic | FR |
|------------|--------|-------------|-----|
| ReorderCalculator | Daily sales rate, current stock, lead time, safety stock days | Score based on (days of stock remaining / (lead time + safety stock)). Lower ratio → higher score | FR-044 |
| TrendCalculator | Sales by period (weekly/monthly) over 3+ months | Trend direction + magnitude. Rising trend with low stock → high score | FR-045 |
| ExpiryRiskCalculator | Batch expiry dates, quantities, selling prices | Days until expiry × quantity at risk × value. Nearer expiry → higher score | FR-046 |
| SlowMovingCalculator | Sales in last 90 days, current stock, purchase price | Zero/low sales with capital tied up → high score | FR-047 |
| OverstockCalculator | Current stock, projected demand (from sales rate), overstock limit | Stock months > overstock limit → score proportional to surplus | FR-048 |
| StockoutCalculator | Current stock, daily sales rate | Projected depletion date. Days until stockout → urgency. Shorter → higher score | FR-049 |

### Data Quality Rule

For any product with < 30 days of sales history: return Recommendation with ConfidenceLevel = 'collecting_data' and a message explaining insufficient history (FR-052). Do not return score-based recommendations.

### Threshold Configuration

Thresholds read from PharmacyConfig (IConfigurationService):
- `recommendation_safety_stock_days` (default: 7)
- `recommendation_lead_time_days` (default: 3)
- `recommendation_overstock_limit` (default: 3 months)
- `recommendation_slow_moving_days` (default: 90)
- `recommendation_expiry_warning_days` (default: 90)
