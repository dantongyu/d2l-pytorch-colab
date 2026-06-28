# Running these notebooks in Google Colab — compatibility status

**TL;DR — As of mid-2026, the first cell of ~115 of the 192 notebooks in this
repo fails on a fresh Google Colab runtime.** The cause is a single line that
nearly every notebook starts with:

```python
!pip install d2l==1.0.3
```

`d2l==1.0.3` **hard-pins ancient dependencies** that are incompatible with the
Python / NumPy / PyTorch that Colab ships today. The notebooks themselves are
fine — only this install line is broken.

---

## Why it breaks (verified, not guessed)

`d2l==1.0.3` declares these exact pins (from its PyPI metadata):

| Pin in `d2l==1.0.3` | Problem on current Colab |
|---|---|
| `numpy==1.23.5`   | **No wheel for Python 3.12** (Colab's Python). pip falls back to a source build, which fails. |
| `pandas==2.0.3`   | Force-downgrades Colab's pandas. |
| `scipy==1.10.1`   | No cp312 wheel / downgrade. |
| `matplotlib==3.7.2` | Downgrades Colab's matplotlib. |

Even on an older Python 3.11 runtime where a `numpy==1.23.5` wheel exists, the
install would **downgrade NumPy below 2.0**, which breaks Colab's *preinstalled*
PyTorch (built against the NumPy 2.x ABI) — so it fails either way, just at a
different step.

**Reproduced on a clean Python 3.12 environment** — `pip install numpy==1.23.5`
(the pin pulled in by `d2l==1.0.3`) aborts during the source build:

```
ERROR: Cannot import 'setuptools.build_meta'   (numpy 1.23.5 has no cp312 wheel)
```

So the break is at **cell 1**, before any deep-learning code even runs.

---

## Status of every chapter

| Chapter | ❌ Breaks (`d2l`) | ✅ Works as-is | ⬚ Empty (no PyTorch port) | 📄 Prose/index |
|---|:--:|:--:|:--:|:--:|
| appendix · mathematics | 10 | 1 |  | 1 |
| appendix · tools | 1 |  |  | 8 |
| attention & transformers | 8 |  |  | 2 |
| builders guide | 3 | 4 |  | 1 |
| computational performance | 5 |  |  | 3 |
| computer vision | 13 | 1 |  | 1 |
| convolutional modern ✅ **fixed** | 0 | 8\* |  | 1 |
| convolutional neural networks ✅ **fixed** | 0 | 5 |  | 2 |
| gaussian processes | 2 |  |  | 2 |
| generative adversarial networks | 2 |  |  | 1 |
| hyperparameter optimization | 5 |  |  | 1 |
| linear classification | 4 |  |  | 4 |
| linear regression | 6 |  |  | 2 |
| multilayer perceptrons | 5 |  |  | 3 |
| NLP · applications | 6 |  |  | 2 |
| NLP · pretraining | 6 | 1 |  | 4 |
| optimization | 11 |  |  | 1 |
| preliminaries | 2 | 5 |  | 1 |
| recommender systems |  |  | 9 | 2 |
| recurrent modern | 7 |  |  | 2 |
| recurrent neural networks | 6 |  |  | 2 |
| reinforcement learning | 2 |  |  | 2 |
| introduction / installation / notation / preface / references / root | | 1 | | 6 |
| **TOTAL (192)** | **104** | **26** | **9** | **53** |

\* The entire **`chapter_convolutional-modern`** chapter (AlexNet, VGG, NiN,
GoogLeNet, batch-norm, ResNet/ResNeXt, DenseNet, RegNet/cnn-design — 8 notebooks)
has been modernized with a shared Colab compatibility-helpers cell and
re-executed end-to-end on Colab-equivalent hardware. `index.ipynb` is prose.
That chapter is the reference for the remaining fixes.

**Legend**
- **❌ Breaks** — runs `!pip install d2l==1.0.3` and/or `from d2l import torch as d2l`. Fails on current Colab.
- **✅ Works as-is** — pure PyTorch, no `d2l` dependency. Runs on Colab today, no changes needed.
- **⬚ Empty** — 0-byte placeholder. These sections (most of *recommender systems*) exist only in D2L's MXNet edition and were never ported to PyTorch; there is nothing to run.
- **📄 Prose/index** — table-of-contents / text-only pages. Render fine; no code.

---

## Notebooks that already work on Colab (no install needed)

These don't touch `d2l`, so they run on a stock Colab runtime as-is:

- `chapter_preliminaries/`: `ndarray`, `pandas`, `linear-algebra`, `autograd`, `lookup-api`
- `chapter_builders-guide/`: `model-construction`, `parameters`, `init-param`, `read-write`
- `chapter_convolutional-neural-networks/padding-and-strides`
- `chapter_computer-vision/rcnn`
- `chapter_natural-language-processing-pretraining/subword-embedding`
- `chapter_appendix-mathematics-for-deep-learning/information-theory`
- `chapter_convolutional-modern/alexnet` (already modernized)

---

## Extra obsolete dependencies (beyond `d2l`) to watch for

A few notebooks layer additional stale installs on top of the `d2l` break:

| Notebook(s) | Extra dependency | Note |
|---|---|---|
| `hyperparameter-optimization/rs-async`, `sh-async` | `syne-tune[extra]` etc. | Still maintained, but the notebooks also need `d2l`. |
| `natural-language-processing-applications/sentiment-analysis-rnn` | `spacy.load('en')` | Deprecated spaCy API; modern spaCy needs `en_core_web_sm`. Optional/markdown. |
| `natural-language-processing-pretraining/bert-dataset` | `nltk.download('punkt')` | Use `punkt_tab` on current NLTK. Optional/markdown. |
| `chapter_installation/index` | `torch==2.0.0 torchvision==0.15.1` | Only stale *instructions* in markdown — don't follow them on Colab; use Colab's preinstalled PyTorch. |

---

## How to fix a broken notebook

The dependency itself is the only problem, so the fix is per-notebook and small.
A fully worked, **tested** example is in
`chapter_convolutional-modern/alexnet.ipynb` (see also its in-notebook change
log). The pattern:

1. **Delete** the `!pip install d2l==1.0.3` cell. Replace it with a lightweight
   setup cell that installs nothing (Colab already ships PyTorch, torchvision,
   matplotlib, NumPy).
2. **Replace** `from d2l import torch as d2l` with a small, self-contained
   *"Colab compatibility helpers"* cell that re-implements **only** the `d2l.*`
   symbols that notebook actually uses (model base classes, `save_hyperparameters`,
   `init_cnn`, `layer_summary`, `FashionMNIST`, `Trainer`, metrics, the live
   loss/accuracy plot, GPU handling), built on modern PyTorch/torchvision, and
   bundle them under a `d2l` namespace so the rest of the notebook is unchanged.
3. Keep all model/architecture/training cells byte-for-byte identical.

Because the API surface differs by chapter, the helpers cell grows or shrinks per
notebook — but it's always small (the AlexNet one is ~150 lines and covers the
common training-loop chapters). Most chapters that use `Module`/`Classifier`/
`Trainer`/`DataModule` can share the *same* helpers cell.

---

## Environment this was validated against

- **Python** 3.12.3
- **PyTorch** 2.11.0 (CUDA 12.8 build) · **torchvision** 0.26.0
- **NumPy** 2.4.4
- GPU: NVIDIA Blackwell (CUDA available); CPU fallback also verified

The modernized AlexNet notebook was executed end-to-end in a fresh kernel
(download → layer summary → 10 GPU epochs → loss/accuracy plot, reaching
~0.84 validation accuracy) with **zero errors**, confirming the fix pattern.
