This repo contains the .ipynb google colab files for submitting to the solona blockchain competition (ended april 30th)

https://www.kaggle.com/datasets/thedevastator/solana-blockchain-dataset
https://www.kaggle.com/competitions/solana-skill-sprint-memcoin-graduation

- basic catboost model on engineered features reached 166th place
- attempts to split eval dataset 25% / use an optuna study actually *decreased* performance significantly
- model from study, however, revealed major bias/confidence on high-activity/volume coins
- v2 includes more engineered features to resolve bias, but requires possibly too much mem