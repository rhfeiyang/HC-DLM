<h1 align="center">Hierarchical Continuous Diffusion Language Models</h1>

<p align="center">
  <a href="https://hc-dlm.github.io/"><img alt="Project page" src="https://img.shields.io/badge/%F0%9F%8C%90%20Project-Page-1f4a8a"></a>
  <img alt="arXiv" src="https://img.shields.io/badge/arXiv-coming%20soon-b31b1b">
  <img alt="Artifacts" src="https://img.shields.io/badge/Artifacts-coming%20soon-d1553c">
</p>

<p align="center">
  <a href="https://rhfeiyang.top/">Hui Ren</a><sup>1</sup>,
  <a href="https://www.linkedin.com/in/zihan-li-616b68325/">Zihan Li</a><sup>1</sup>,
  <a href="https://ruachang.github.io/">Chang Liu</a><sup>1</sup>,
  <a href="https://harryliew.github.io/">Huidong Liu</a><sup>2</sup>,
  <a href="https://www.alexander-schwing.de/">Alexander Schwing</a><sup>1</sup><br>
  <sup>1</sup>University of Illinois Urbana-Champaign &nbsp;&nbsp; <sup>2</sup>Amazon.com, Inc.
</p>

<p align="center"><i><b>HC-DLM</b> is a diffusion language model whose only persistent state is a continuous latent:<br>tokens are read out of it at every step and fed back as the scaffold for the next.</i></p>

<p align="center">
  <a href="https://hc-dlm.github.io/"><img src="assets/demo.gif" alt="One HC-DLM reverse trajectory on a sentence: the continuous latent is denoised step by step; at every step the whole sentence is read out of it, re-noised, and fed back as the scaffold for the next latent update, so early words can still be revised." width="860"></a>
</p>
<p align="center"><sub>One reverse trajectory on a sentence. Scripted illustration of the sampler, not model output. Interactive version on the <a href="https://hc-dlm.github.io/">project page</a>.</sub></p>

> [!NOTE]
> This repository hosts the project description for now. Code and artifacts are being prepared for release. **Watch** or **star** the repo to be notified.

<p align="center">
  <a href="#-idea">Idea</a> ·
  <a href="#-results">Results</a> ·
  <a href="#-release">Release</a> ·
  <a href="#-citation">Citation</a>
</p>

## 💡 Idea

Both families of diffusion language models leave something on the table:

- **Discrete diffusion** decodes in parallel, but samples every token from its own marginal.
- **Continuous diffusion** denoises one shared state, but nothing ties that state to a valid token sequence until the end.

HC-DLM makes the reverse process a **hierarchy**. The continuous latent is the only state that persists across steps; the tokens are read out of it at every step and fed back as the scaffold for the next latent update.

$$
p_{\theta,\phi}(k_{0:T},x_{0:T})=p(x_T)\,p_\theta(k_0\mid x_0)\prod_{t=1}^{T}\underbrace{p_\theta(k_t\mid x_t)}_{\text{read out}}\;\underbrace{p_\phi(x_{t-1}\mid x_t,k_t)}_{\text{feed back}}
$$

Each reverse step does three things:

<table>
  <tr>
    <th align="center" width="33%">① Denoise</th>
    <th align="center" width="33%">② Read out</th>
    <th align="center" width="33%">③ Re-noise</th>
  </tr>
  <tr>
    <td align="center">Conditioned on the current token scaffold, the denoiser estimates the clean latent and advances the latent one step.</td>
    <td align="center">The token predictor decodes the whole sequence from that estimate, so parallel tokens share one cause.</td>
    <td align="center">The known forward kernel corrupts the read-out tokens to the next level; nothing is ever frozen.</td>
  </tr>
</table>

The tokens have no transition chain of their own, which separates HC-DLM from hybrid models that attach a continuous signal to a self-contained discrete chain. A single variational bound on the token likelihood trains the encoder, the denoiser and the token predictor together.

<p align="center">
  <img src="assets/training.webp" alt="Training pipeline: an encoder maps clean tokens to a latent, independent forward kernels produce the noisy pair, the token-conditioned denoiser predicts the clean latent, and the token predictor decodes the latent back to tokens." width="680">
</p>
<p align="center"><sub>Training: an encoder maps clean tokens to a latent, independent forward kernels produce the noisy pair, the token-conditioned denoiser predicts the clean latent, and the token predictor decodes it back to tokens.</sub></p>

## 📊 Results

Structured reasoning (Sudoku), mathematical planning (Countdown) and language modeling (LM1B), against discrete, continuous and hybrid diffusion baselines at matched model size.

<p align="center">
  <img src="assets/results.webp" alt="Hard Sudoku accuracy 72.41% at 6M parameters, 1.68 over CCDD and 22.53 over masked diffusion; Countdown CD5 accuracy 37.52% at 6M, 12.17 over CCDD; LM1B generative perplexity 75.5, 1.8 lower than Plaid and 16.7 lower than LangFlow." width="860">
</p>

<p align="center">
  <img src="assets/analysis.webp" alt="Two line charts. Left: Sudoku accuracy of the decoded intermediate prediction along the trajectory rises earlier and higher for HC-DLM than for a latent diffusion model without the token scaffold. Right: Hard Sudoku accuracy against the number of denoising steps stays high for HC-DLM while MDM degrades as steps are reduced." width="760">
</p>
<p align="center"><sub><b>Left:</b> the scaffold locks in structure early: accuracy of the intermediate prediction along the trajectory, vs. latent diffusion without it. <b>Right:</b> robust to fewer steps: Hard Sudoku accuracy vs. number of denoising steps, vs. MDM.</sub></p>

<details>
<summary><b>📋 Full tables</b> (accuracy %; generative perplexity, lower is better)</summary>
<br>

**Sudoku & Countdown** (6M parameters, accuracy %)

| Method                  | Sudoku Easy | Sudoku Hard |       CD4 |       CD5 |
|-------------------------|------------:|------------:|----------:|----------:|
| MDM (top-prob. margin)  |       89.49 |       49.88 |      50.8 |      21.3 |
| CCDD                    |   **94.65** |       70.73 |     81.18 |     25.35 |
| **HC-DLM**              |       94.21 |   **72.41** | **84.41** | **37.52** |

**LM1B** (generative perplexity)

| Method     | Params | Gen. PPL ↓ |
|------------|-------:|-----------:|
| MDM        |   116M |      103.9 |
| Duo        |   116M |       97.6 |
| LangFlow   |   117M |       92.2 |
| Plaid      |   109M |       77.3 |
| **HC-DLM** |   118M |   **75.5** |

**Ablation:** removing either level falls well short of the full model (Hard Sudoku accuracy %).

| Variant                                 | Hard Sudoku |
|-----------------------------------------|------------:|
| Continuous only (no token scaffold)     |       24.74 |
| Discrete only (purely discrete chain)   |       49.88 |
| **HC-DLM** (both levels)                |   **72.41** |

</details>

## 📦 Release

The code and the artifacts are coming soon. Stay tuned!

## 📖 Citation

If you find this work useful, please consider citing:

```bibtex
@article{ren2026hierarchical,
  title   = {Hierarchical Continuous Diffusion Language Models},
  author  = {Ren, Hui and Li, Zihan and Liu, Chang and Liu, Huidong and Schwing, Alexander},
  journal = {arXiv preprint arXiv:XXXX.XXXXX},
  year    = {2026}
}
```
