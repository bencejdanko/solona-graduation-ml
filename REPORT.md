1.  **Model Performance & Errors:**
    *   The model is reasonable (PR AUC ~0.40) but struggles with F1 score (~0.47), indicating difficulty in setting a good threshold or distinguishing borderline cases.
    *   **High False Negatives (FNs):** The model misses slightly more positive cases (983 FNs) than it correctly identifies (868 TPs). This is a major area for improvement.
    *   **Significant False Positives (FPs):** There are many cases (1866 FPs) where the model predicts graduation, but it doesn't happen.
    *   **Probability Distributions:** Both FNs and FPs have predicted probabilities clustered near the 0.5 threshold, confirming they are hard-to-classify cases. FNs are generally predicted with low probability (<0.3 mean), while FPs are predicted just above the threshold (0.68 mean, 25th percentile at 0.57).

2.  **Feature Importance:**
    *   **Dominance of LP State:** `lp_sol_end_100s` and `lp_token_end_100s` are paramount, as expected. Liquidity is key.
    *   **Fees Matter:** `avg_fee_100s` and `total_fee_100s` are surprisingly high in importance. This might indicate urgency, bot activity, or sophisticated users willing to pay for inclusion.
    *   **Volume & Price:** Various volume (`buy_sol_vol_100s`, `total_token_vol_100s`, `net_sol_vol_100s`) and price-related (`price_end_100s`, `avg_price_per_buy_tx_100s`, `price_change_pct_100s`) features are important.
    *   **Low Importance Counts:** Basic transaction counts (`tx_count_100s`, `sell_tx_count_100s`) and seller-specific features (`sell_sol_vol_100s`, `unique_sellers_count_100s`) have low importance, suggesting aggregate volume and net changes are more informative than raw counts alone.
    *   **Useless Feature:** `first_slot_offset_100s` has zero importance, confirming activity almost always starts immediately.

3.  **Detailed Feature Distributions (The Most Actionable Part):**
    *   **FN Profile (Missed Graduations):** Compared to True Positives (TPs), FNs consistently show:
        *   *Lower* transaction counts, SOL/token volumes (total, buy, net), LP SOL/token end values, LP changes (absolute & percentage), price changes, various rates (tx/sec, SOL/sec, LP/sec, wallets/sec).
        *   The distributions often show limited overlap, with the 75th percentile of FNs being below the 25th percentile of TPs for key metrics like `lp_sol_end_100s`, `net_sol_vol_100s`, `price_change_pct_100s`. **FNs look like tokens that start but fail to gain significant traction/liquidity/price momentum within the window.**
        *   Interestingly, FNs tend to have *longer* activity spans (`slot_span_100s`, `last_slot_offset_100s`, `time_span_seconds_100s`), perhaps indicating slow, drawn-out activity rather than a decisive burst.
    *   **FP Profile (False Alarms):** Compared to True Negatives (TNs), FPs consistently show:
        *   *Higher* transaction counts, SOL/token volumes (total, buy, sell, net), unique wallet counts, LP values, LP changes, price changes, and rates.
        *   The distributions often show limited overlap (FP P25 > TN P75). **FPs look like highly active tokens that *should* graduate based on raw activity but don't.** The model correctly identifies high activity but incorrectly classifies it as *successful* activity.
    *   **FP vs. TP Confusion:** For many features, especially counts, volumes, and rates, the *medians and interquartile ranges (IQRs) of FPs and TPs significantly overlap*. This is the crux of the classification difficulty. **FPs often mimic the *magnitude* of activity seen in TPs.** The difference likely lies in the *efficiency*, *timing*, *volatility*, or *sustainability* of that activity.

**Action Plan: Refining Feature Engineering**

Based on the analysis, we need features that better capture:

1.  **Momentum & Acceleration (Distinguish FNs from TPs):** FNs lack the "take-off."
2.  **Efficiency & Sustainability (Distinguish FPs from TPs):** FPs seem active but perhaps inefficiently or unsustainably.
3.  **Volatility & Churn (Potentially identify FPs):** High activity might be bot churn or pumps followed by dumps.
4.  **Fee Behavior (Leverage important feature):** Explore fee patterns further.

**Integrating New Features into the Script:**

We'll modify the existing script (`memcoin_no_visuals.ipynb`) in three places:
*   **Step 2: Initialize Feature Storage:** Add new keys to the `features_per_mint` dictionary template.
*   **Step 3: Process Chunks Incrementally:** Update the logic within the `for row in df_chunk.itertuples():` loop to accumulate necessary data for the new features (e.g., store lists of SOL amounts for standard deviation, track values at different time points). *Caution: Storing lists can increase memory usage significantly.*
*   **Step 4: Finalize Features:** Calculate the new derived features after the loop finishes.

**Proposed New Feature Categories & Examples:**

**(A) Momentum & Acceleration Features:**

*   **Idea:** Compare activity/liquidity in the first half vs. second half of the 100-slot window.
*   **How:**
    *   In the loop, track sums/counts/last values specifically for slots `target_start` to `target_start + 49` and `target_start + 50` to `target_end - 1`.
    *   **New Features:**
        *   `sol_vol_ratio_half_100s`: (Volume in 2nd half) / (Volume in 1st half)
        *   `lp_sol_change_ratio_half_100s`: (LP SOL change in 2nd half) / (LP SOL change in 1st half)
        *   `unique_wallets_ratio_half_100s`: (New wallets in 2nd half) / (New wallets in 1st half)
        *   `tx_rate_increase_100s`: (Tx per slot in 2nd half) - (Tx per slot in 1st half)
*   **Why:** TPs might show acceleration in the second half, while FNs might fizzle out.

**(B) Efficiency & Sustainability Features:**

*   **Idea:** Relate the SOL input to the outcome (LP SOL increase, price change).
*   **How:** Use existing aggregated values.
*   **New Features:**
    *   `lp_sol_per_buy_sol_100s`: `lp_sol_change_100s / buy_sol_vol_100s` (How much LP SOL gained per SOL bought?)
    *   `net_sol_efficiency_100s`: `lp_sol_change_100s / net_sol_vol_100s`
    *   `price_change_per_net_sol_100s`: `price_change_pct_100s / net_sol_vol_100s`
    *   `sell_pressure_ratio_100s`: `sell_sol_vol_100s / total_sol_vol_100s` (Already somewhat captured by `buy_sell_sol_vol_ratio`, but direct ratio might be useful).
    *   `lp_sol_sustainability_100s`: `lp_sol_end_100s / max_lp_sol_100s` (Did the LP SOL end near its peak, or did it drop off?)
*   **Why:** FPs might have high volume but low efficiency (lots of buys offset by sells, or poor price impact), or show a drop-off in LP SOL towards the end.

**(C) Volatility & Churn Features:**

*   **Idea:** Measure the standard deviation of key metrics. *Requires storing lists during chunk processing, increasing memory usage.*
*   **How:**
    *   In the loop, append values like `quote_coin_amount`, `virtual_sol_balance_after`, `virtual_token_balance_after` to lists within `features_per_mint[mint]`.
    *   In finalization, calculate standard deviations.
    *   **New Features:**
        *   `stddev_sol_vol_tx_100s`: Std dev of `quote_coin_amount` per transaction.
        *   `stddev_lp_sol_100s`: Std dev of `virtual_sol_balance_after`.
        *   `stddev_lp_token_100s`: Std dev of `virtual_token_balance_after`.
        *   `stddev_price_100s`: Calculate implied price per transaction (`virtual_sol_balance_after / virtual_token_balance_after`) and find its std dev.
        *   `wallet_churn_ratio_100s`: `tx_count_100s / unique_wallets_count_100s` (already calculated as `avg_tx_per_wallet_100s`, maybe rename for clarity?) - High values might indicate bots.
*   **Why:** High volatility might characterise FPs (pump and dump dynamics).

**(D) Enhanced Fee/Gas Features:**

*   **Idea:** Normalize fees or look at gas efficiency.
*   **How:** Use existing aggregated values.
*   **New Features:**
    *   `avg_fee_per_sol_100s`: `total_fee_100s / total_sol_vol_100s`
    *   `avg_gas_efficiency_100s`: Requires `provided_gas_limit`. Calculate `avg(consumed_gas / provided_gas_limit)` if `provided_gas_limit` is added to `COLS_TO_LOAD_FROM_CHUNKS`. (Already included in list, check if used). *Yes, it's loaded.* Need to store consumed/limit per tx or calculate average efficiency in finalization.
*   **Why:** Relate fees to economic activity; gas efficiency might indicate bot sophistication or network conditions.

**(E) Time-Weighted Features (Advanced):**

*   **Idea:** Give more weight to recent activity within the 100 slots.
*   **How:** Requires calculating weights based on `slot - target_start` during the loop and applying them to sums/averages. More complex implementation.
*   **Why:** Recent trends might be more predictive.

**Implementation Steps:**

1.  **Backup:** Save your current `memcoin_no_visuals.ipynb` and `engineered_features_100s_v1.parquet`.
2.  **Modify `COLS_TO_LOAD_FROM_CHUNKS`:** Ensure `provided_gas_limit` is definitely being loaded if you want `avg_gas_efficiency_100s`.
3.  **Modify Feature Initialization (Step 2):** Add placeholders for the new features you choose to implement (e.g., `sol_vol_first_half`: 0.0, `sol_vol_second_half`: 0.0, `tx_sol_amounts_list`: []).
4.  **Modify Chunk Processing Loop (Step 3):**
    *   Add logic to check `slot` relative to `target_start + 50` for half-window features.
    *   If doing std dev features, append relevant values (`row.quote_coin_amount`, etc.) to the lists initialized in Step 2. Handle NaNs appropriately before appending.
5.  **Modify Feature Finalization (Step 4):**
    *   Calculate the derived features (ratios, efficiencies, std devs from lists, etc.).
    *   **Crucially handle division by zero:** Replace resulting `inf` or `-inf` with `np.nan` or a suitable large/small number consistently.
    *   Handle empty lists if calculating std dev (result should be 0 or NaN).
    *   Remove any temporary lists (like `tx_sol_amounts_list`) before creating the final DataFrame.
6.  **Save New Features:** Save the updated DataFrame to a *new* file (e.g., `engineered_features_100s_v2.parquet`).
7.  **Data Cleaning for Model:** Before training:
    *   Drop the useless `first_slot_offset_100s`.
    *   Decide on NaN imputation: CatBoost can handle NaNs internally (often a good default), or you can impute with median, mean, or zero beforehand. Be consistent. Check features like `buy_sell_tx_ratio` which had NaNs in the original analysis.
    *   Drop datetime columns (`first_time_100s`, `last_time_100s`) or convert them to numerical representations (e.g., epoch seconds) if not already done implicitly by saving/loading.
    *   Drop the `mint` column before training.
8.  **Retrain & Evaluate:**
    *   Load the *new* features (`v2`).
    *   Merge with the target variable (`train.csv`).
    *   Perform the *exact same* train/validation split (`random_state=42`, `stratify=y`).
    *   Retrain the CatBoost model using the *same* best hyperparameters found previously. *Do not* re-run Optuna yet. The goal is to see if the *new features* improve performance with the *same* model settings.
    *   Evaluate using the same metrics (LogLoss, ROC AUC, PR AUC, F1, Classification Report, Confusion Matrix). Compare directly to the previous results.
    *   Re-examine feature importances and the detailed error analysis on the *new* results.

**Incorporating Insights into Model Generation:**

*   **Feature Selection:** After evaluating the model with new features, check their importances. If some new features are highly important and improve metrics, keep them. If some remain low importance or don't help/hurt performance, consider removing them to simplify the model.
*   **Threshold Tuning:** The high FN/FP count near the 0.5 threshold suggests tuning might help. After training the *final* model on the best feature set, use the validation set predictions (`val_preds_proba`) to find the threshold that optimizes your primary metric (e.g., F1 score or a custom metric balancing FNs and FPs). Calculate F1 scores across a range of thresholds (e.g., 0.1 to 0.9) and pick the best one for generating final predictions on the test set.
*   **Potential Re-Tuning:** If the new features drastically change the data distribution or model behavior, a *limited* re-run of Optuna focusing on `learning_rate` and `scale_pos_weight` might be warranted, but try improving features first.

Start by implementing a subset of the proposed features, focusing on **Momentum/Acceleration** and **Efficiency/Sustainability** as they directly address the observed FN/FP profiles. Avoid the memory-intensive volatility features initially unless memory constraints aren't an issue.