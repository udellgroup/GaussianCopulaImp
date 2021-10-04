# GaussianCopulaImp
This package provides a python implemention to fit a Gaussian copula model, on continuous, ordinal and binary mixed data with missing values. The user could either fit a full rank Gaussian copula model [1] or a low rank Gaussian copula model [2], which builds upon the PPCA model in a latent space. The package also includes the mini-batch and online implementation of the full rank Gaussian copula model [3], which can accelerate the training process and adapt to a changing distribution in the streaming data. The fitted model can be used for missing value imputation, latent correlation learning and latent correlation change detection.

## Installation

The easiest way is to install using pip: `pip install GaussianCopulaImp` 

## Overview

There are two different models available for use: the standard Gaussian copula model and the low rank Gaussian copula model. When working with skinny datasets (small p), the standard Gaussian copula model is preferred. In contrast, the low rank Gaussian copula model is recommended when working with wide datasets (large p). 

There are three training options for the standard Gaussian copula model: standard offline training, mini-batch offline training and mini-batch online training. In short, mini-batch offline training is often much faster than the standard offline training, by using more frequent model updates. Online training is designed for the streaming data scenario when data comes  at different time points or the data distribution is changing over time. Parallelism is now supported for all training options with the standard Gaussian copula model for further acceleration. 

For low rank Gaussian copula model, only standard offline training without parallelism is supported at this moment. Parallelism will be supported soon. The development of mini-batch training (both offline and online) is nontrivial. Please contact the authors if you are interested in collaboration for developing those functionalities.

Please also see below for more detailed dicussions on how to select the model and training option that works best for your purpose.

## Select the right model to use
Both the standard Gaussian copula model (GC for short) and the low rank Gaussian copula model (LRGC for short) estimate a copula correlation matrix of size p by p for p variables. The difference is that LRGC imposes a low rank structure on the latent correlation matrix such that the number of free parameters is O(pk), with a specified low rank k. In contrast, GC has O(p^2) free parameters.

GC is recommended in the classical setting where the number of samples is much larger than the number of variabels, i.e. n>>p. 
For high-dimensional setting (p>n or large p), LRGC is recommended because (1) the estimated copula correlation matrix will be singular when p>n; (2) the O(np^3) time complexity of GC may be too expensive for large p, while LRGC only has O(npk^2) time complexity with a small specified rank k.

Most of our experiments have at least around 200 samples points (n>200). There may be some problematic behavior when n is very small, since the copula marginal estimation accuracy depends on n. Please report any bugs you encounter!

## Select the right training option
The standard offline training is an approximate EM algorithm, the simplest one to use. Mini-batch offline training is an online EM algorithm, which can greatly accelerate the training. Mini-batch online training differs with mini-batch offline training only in how many data points they remember. The offline version remebers all the data to improve the copula marginal estimation accuracy, while the online version only remembers recent data in a lookback window (length determined by the user).

When the computation time is the only consideration, mini-batch offline training is prefered. However, there are situations that remembering all data points is either impossible or undesirable. For example, when the data comes at different times and a model is required at each arrival time, future data is unavailable and remembering all data is thus impossible. Also, when the data distribution changes over time, remebering all data points includes much noise in historical data that no longer helps explaining current data and thus undesirable. For these situations, mini-batch online training should be used.

## Examples 
```
from GaussianCopulaImp.gaussian_copula import GaussianCopula
from GaussianCopulaImp.helper_data_generation import generate_sigma, generate_mixed_from_gc
from GaussianCopulaImp.helper_evaluation import get_smae, get_scaled_error
from GaussianCopulaImp.helper_mask import mask_types
import numpy as np
seed = 101

# generate 15-dim mixed data (5 exponential variables, 5 1-5 ordinal variables and 5 boolean variables) 
copula_corr = generate_sigma(seed=seed, p=15)
X = generate_mixed_from_gc(sigma=copula_corr, n=2000, seed=seed)

# randomly 2 (out of 5) entries of each variable type in each row
X_mask = mask_types(X, mask_num=2, seed=seed)
print('The first row of the masked dataset: ')
print(X_mask[0,:])

# model fitting 
gc = GaussianCopula()
out = gc.impute_missing(X=X_mask, verbose=True)
X_imp, copula_corr_est = out['imputed_data'], out['copula_corr']

# Evaluation: compute the scaled-MAE (SMAE) for each data type (scaled by MAE of median imputation) 
smae = get_smae(X_imp, X, X_mask)
print(f'The SMAE across 5 exponential variables has: mean {smae[:5].mean():.3f} and std {smae[:5].std():.3f}')
print(f'The SMAE across 5 1-5 oridnal variables has: mean {smae[5:10].mean():.3f} and std {smae[5:10].std():.3f}')
print(f'The SMAE across 5 boolean variables has: mean {smae[10:].mean():.3f} and std {smae[10:].std():.3f}')
# Evaluation: compute the scaled l2 error of the estiamted copula correlation matrix (scaled by the l2 norm of the true copula correlation) 
cor_error = get_scaled_error(copula_corr_est, copula_corr)
print(f'The scaled correlation error is: {cor_error:.3f}')
```



## References
[1] Zhao, Y. and Udell, M. Missing value imputation for mixed data via Gaussian copula, KDD 2020.

[2] Zhao, Y. and Udell, M. Matrix completion with quantified uncertainty through low rank Gaussian copula, NeurIPS 2020.

[3] Zhao, Y., Landgrebe, E., Shekhtman E., and Udell, M. Online missing value imputation and correlation change detection for mixed-type Data via Gaussian Copula, arXiv 2020.
