# 🪙 Meme Coin Graduation Predictor

This repository contains Google Colab notebooks and modeling code for the **Solana Skill Sprint: Memecoin Graduation** competition (ended **April 30th, 2025**). The goal was to predict whether a newly launched meme coin would "graduate" (≥ 85 SOL in liquidity) using on-chain behavior data from its first 100 Solana blocks (~50 seconds).

---

## 📂 Dataset Sources

- **Official Competition Dataset**:  
  🔗 [Solana Skill Sprint: Memcoin Graduation (Kaggle Competition)](https://www.kaggle.com/competitions/solana-skill-sprint-memcoin-graduation)

- **Raw Behavioral Data**:  
  🔗 [Solana Blockchain Dataset (Kaggle Datasets)](https://www.kaggle.com/datasets/dremovd/pump-fun-graduation-february-2025)

---

## 📊 Dataset Structure

### `train.csv` / `test_unlabeled.csv`
- One row per token (`mint`)
- `has_graduated`: binary label (target variable in `train.csv`, absent in test)

### `chunk_*.csv`
- Engineered behavioral features within first 100 blocks
- Include: 
  - Trading metrics (`buyer_count`, `mean_price`, `total_volume`)
  - Wallet behaviors (`unique_wallets`, `jito_activity`)
  - Temporal metrics and volatility proxies

### `dune_token_info.csv`
- Supplemental token metadata
- Includes tags like airdrop, token type, pool starting balance

---

## 🧠 Model Overview

### 🔹 Baseline:
- **CatBoostClassifier** using concatenated `chunk_*.csv` features
- Achieved **166th place** on public leaderboard

### 🔹 Experimentation:
- 25% validation split + **Optuna** study introduced significant overfitting
- Revealed strong **confidence bias** on tokens with very high activity
- SHAP analysis confirmed model overly favored volume-based features

### 🔹 v2 Model:
- Incorporated additional engineered features targeting:
  - Early whale buys
  - Wallet churn
  - Activity velocity
- Partially resolves bias but:
  - **Memory usage spiked**
  - Training became unstable on free-tier runtimes

---

## 🧪 Key Learnings

- **Early behavioral data is extremely noisy**
- **Feature selection** is more impactful than hyperparameter tuning
- Models overconfidently graduate high-activity tokens unless normalized
- Coin launch metadata can **contextualize behavior** and help mitigate bias

---

## 🛠 Notebooks

- `Meme_Coin_Model_From_Engineered_Features.ipynb`: core modeling notebook
- 'lightgbm-solana-model-1.ipynb': initial LightGBM model
- Includes loading, feature engineering, baseline CatBoost modeling, and leaderboard submission code

---

## 📤 Submission Format

The final `submission.csv` includes:
- `mint`: token identifier
- `has_graduated`: predicted probability (float in [0, 1])