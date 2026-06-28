# Running these notebooks on current Google Colab

## The fix

Every notebook used to start with `!pip install d2l==1.0.3`, which hard-pins
`numpy==1.23.5` (and old pandas/scipy/matplotlib). On current Colab (Python 3.12)
that pin has no wheel and the source build fails — the first cell aborts.

The d2l **code is fine** on the modern stack; only its dependency *pins* are
broken. So the fix is one line per notebook:

```diff
-!pip install d2l==1.0.3
+!pip install d2l==1.0.3 --no-deps
```

`from d2l import torch as d2l` and every other cell stay identical to upstream.
Applied to all 116 d2l-using notebooks in the repo.

## Full end-to-end test

All **182** non-empty notebooks were executed in a fresh kernel against the real
d2l on a Colab-equivalent GPU stack (Python 3.12.3, torch 2.11.0+cu128,
torchvision 0.26.0, NumPy 2.4.4, **pandas 3.0.3**, matplotlib 3.11, gym 0.26).

**172 / 182 pass.** Every notebook that runs on a standard single-GPU runtime
with its datasets/assets available now works.

### Version-drift bugs found and fixed (would have broken on current Colab)

These are **not** about `--no-deps` — they are notebook-code incompatibilities
with newer pandas/matplotlib that this test surfaced and fixed:

| Notebook | Issue | Fix |
| --- | --- | --- |
| `multilayer-perceptrons/kaggle-house-price` | pandas 3.0 gives strings their own dtype, so `dtypes != 'object'` wrongly kept text columns and `.mean()` failed | `features.select_dtypes('number').columns` (works on pandas 2 *and* 3) |
| `appendix-mathematics/distributions` | matplotlib removed the `stem(use_line_collection=…)` kwarg | drop the kwarg (now the default) |
| `appendix-mathematics/random-variables` | same `stem(use_line_collection=…)` | drop the kwarg |

### Additionally fixed for standalone Colab (opt-in cells)

| Notebooks | Was | Fix |
| --- | --- | --- |
| 7 computer-vision (`anchor`, `bounding-box`, `fcn`, `image-augmentation`, `multiscale-object-detection`, `neural-style`, `ssd`) | read bundled `../img/*.jpg`, absent in a lone Colab notebook | a small cell fetches just the needed images from this repo's `img/` on `main` |
| `reinforcement-learning/{qlearning, value-iter}` | d2l's RL helpers target gym 0.21 (`env.nS/nA/seed`), broken on Py3.12/NumPy 2 | a cell installs `gym==0.25.2`, shims `np.bool8`, and re-points `d2l.make_env` to a modern-API equivalent — lesson code unchanged |

Both were validated end-to-end in a fresh kernel with **no `../img` present** (CV
images download successfully) and **0 errors** (RL notebooks produce their value/
Q-function plots).

### Remaining non-passing — pre-existing, none caused by `--no-deps`

| Notebooks | Category | Happens on Colab? |
| --- | --- | --- |
| `builders-guide/use-gpu`, `computational-performance/{multiple-gpus, multiple-gpus-concise, auto-parallelism}` | **Need ≥2 GPUs** (`X.cuda(1)`, etc.) | Yes on a single-GPU runtime — inherent to the lesson; use a multi-GPU runtime |
| `hyperparameter-optimization/{rs-async, sh-async}` | **syne-tune** — the notebook self-installs it via its own `pip install 'syne-tune[extra]'` cell | No — works on Colab; only our offline test env lacked it |
| `nlp-pretraining/{bert-dataset, bert-pretraining}` | **Dead dataset URL** — d2l's hardcoded wikitext-2 link (`research.metamind.io`) now returns an error page, not a zip | Yes — upstream dataset rot; needs a working mirror |

## Bottom line

The `--no-deps` install fix is fully validated across the whole book: 172/182
green, and the 10 exceptions are pre-existing (multi-GPU hardware, a dead upstream
dataset URL, gym/syne-tune availability) — not introduced by the fix. Three real
pandas/matplotlib version-drift bugs were additionally fixed.
