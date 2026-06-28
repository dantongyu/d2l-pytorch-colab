# Running these notebooks on current Google Colab

## The problem

Almost every notebook starts with:

```python
!pip install d2l==1.0.3
```

`d2l==1.0.3` **hard-pins** `numpy==1.23.5`, `pandas==2.0.3`, `scipy==1.10.1`,
`matplotlib==3.7.2`. On a current Colab runtime (Python 3.12) `numpy==1.23.5`
has no wheel, so pip tries to build it from source and fails — the very first
cell aborts. Even where a pin resolves, it downgrades NumPy below what Colab's
preinstalled PyTorch needs. So the install breaks before any code runs.

## The fix (minimal, one line per notebook)

The d2l **code is fine** on the modern stack — only its dependency *pins* are
broken. So we install d2l without letting it drag in those pins, and rely on the
modern NumPy / pandas / PyTorch that Colab already ships:

```python
!pip install d2l==1.0.3 --no-deps
```

That is the **only** change. `from d2l import torch as d2l` and every other cell
stay byte-for-byte identical to the original notebook. Verified: the real d2l
imports and runs cleanly under Python 3.12 / NumPy 2.x / torch 2.x, with
`Trainer`, `Vocab`, `RNN`, `MultiHeadAttention`, etc. all working.

> Note: `--no-deps` relies on Colab having d2l's runtime libraries preinstalled
> (numpy, pandas, scipy, requests, pillow, matplotlib, matplotlib-inline,
> IPython, torch, torchvision). Stock Colab has all of them.

## End-to-end test results — Chapters 1–12

Every notebook in chapters 1–12 (96 notebooks) was executed in a fresh kernel
against the real d2l on a Colab-equivalent stack (Python 3.12.3, torch
2.11.0+cu128, torchvision 0.26.0, NumPy 2.4.4, GPU). **94 / 96 passed.**

| Chapter | Pass |
|---|---|
| 1 · introduction | 1/1 |
| 2 · preliminaries | 8/8 |
| 3 · linear-regression | 8/8 |
| 4 · linear-classification | 8/8 |
| 5 · multilayer-perceptrons | 7/8 |
| 6 · builders-guide | 7/8 |
| 7 · convolutional-neural-networks | 7/7 |
| 8 · convolutional-modern | 9/9 |
| 9 · recurrent-neural-networks | 8/8 |
| 10 · recurrent-modern | 9/9 |
| 11 · attention-mechanisms-and-transformers | 10/10 |
| 12 · optimization | 12/12 |
| **Total** | **94/96** |

### The two non-passing notebooks (neither caused by `--no-deps`)

1. **`chapter_builders-guide/use-gpu.ipynb`** — contains `Z = X.cuda(1)`, which
   copies a tensor to a *second* GPU. It fails on any single-GPU machine,
   including a standard single-GPU Colab runtime. This is inherent to the
   notebook (the book demonstrates it on a 2-GPU host); the original has the same
   behavior. Run it on a multi-GPU runtime, or skip the multi-GPU cells.

2. **`chapter_multilayer-perceptrons/kaggle-house-price.ipynb`** — hits a
   **pandas 3.0** breaking change: pandas 3.0 gives string columns their own
   dtype, so the notebook's `features.dtypes != 'object'` mask wrongly includes
   text columns and `.mean()` then fails on strings. This only occurs if the
   runtime has pandas ≥ 3.0 (our test env had pandas 3.0.3). On Colab's pandas
   2.x it passes. If your Colab is on pandas 3.0, the one-line fix is to select
   numeric columns by kind instead of by `'object'`:
   `numeric_features = features.select_dtypes('number').columns`.

## Scope

`--no-deps` has been applied to all 64 d2l-using notebooks in chapters 1–12.
Index/prose notebooks need no change. The same one-line fix works for the
remaining chapters (13+) and can be rolled out the same way.
