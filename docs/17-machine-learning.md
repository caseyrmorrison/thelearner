# Machine Learning — Concepts and the Math Underneath

A refresher for someone who has seen this material before and wants the *why* and the
*math* back, not a first pass. It builds from "what is learning, formally" up through
backpropagation and attention, deriving the load-bearing equations rather than asserting
them. It pairs with **[Nabla](../project-9-nabla/)**, a from-scratch automatic-
differentiation engine — §5's backprop math *is* Nabla's source code, so read them
together.

> Math renders on GitHub (LaTeX in Markdown). In a plain editor the `$…$` and `$$…$$` are
> raw — still legible, just unrendered.

---

## 1. What Learning Is, Formally

Supervised learning assumes data is drawn i.i.d. from an unknown joint distribution
$p(x, y)$, and seeks a function $f_\theta: \mathcal{X} \to \mathcal{Y}$, parameterized by
$\theta$, that predicts $y$ from $x$. "Learning" is choosing $\theta$. Three ingredients
define every problem:

- **A model** $f_\theta$ — the hypothesis class you'll search (linear maps, trees, neural
  nets). This choice bounds what's even representable.
- **A loss** $L(\hat{y}, y)$ — the cost of predicting $\hat{y}$ when the truth is $y$. It
  encodes what "wrong" means for your task.
- **An optimizer** — the procedure that adjusts $\theta$ to reduce loss.

The goal is *not* to fit the training data — it's to minimize **expected risk** over the
true distribution:

$$R(\theta) = \mathbb{E}_{(x,y)\sim p}\big[L(f_\theta(x), y)\big]$$

But $p$ is unknown; you only have samples. So you minimize **empirical risk**, the average
loss over your $n$ training points, usually plus a regularizer $\Omega$:

$$\hat{R}(\theta) = \frac{1}{n}\sum_{i=1}^{n} L(f_\theta(x_i), y_i) + \lambda\,\Omega(\theta)$$

Everything else in this document is machinery for (a) choosing $L$, (b) minimizing
$\hat{R}$, and (c) ensuring low $\hat{R}$ actually implies low $R$ — the **generalization**
problem, which is the whole game. The gap between them is the difference between memorizing
and learning.

The other settings in one line each: **unsupervised** learning has no $y$ (find structure —
clustering, density estimation, dimensionality reduction); **reinforcement** learning
replaces $(x,y)$ pairs with a reward signal from an environment you act in, and must handle
credit assignment over time.

## 2. Where Loss Functions Come From (Maximum Likelihood)

Losses aren't arbitrary — the standard ones fall out of **maximum likelihood estimation**.
Assume your model defines a conditional distribution $p_\theta(y \mid x)$. The likelihood of
the data is $\prod_i p_\theta(y_i \mid x_i)$; maximizing it is equivalent to minimizing the
**negative log-likelihood**:

$$\theta^* = \arg\max_\theta \prod_i p_\theta(y_i\mid x_i) = \arg\min_\theta \sum_i -\log p_\theta(y_i\mid x_i)$$

The two most common losses are exactly this for two choices of $p_\theta$:

**Regression → Mean Squared Error.** Assume $y = f_\theta(x) + \varepsilon$ with Gaussian
noise $\varepsilon \sim \mathcal{N}(0, \sigma^2)$. Then
$-\log p_\theta(y\mid x) = \frac{(y - f_\theta(x))^2}{2\sigma^2} + \text{const}$, and summing
over the data gives, up to constants,

$$L_{\text{MSE}} = \frac{1}{n}\sum_i \big(f_\theta(x_i) - y_i\big)^2.$$

So **MSE is the maximum-likelihood loss under Gaussian noise** — that's the assumption you
silently make every time you use it.

**Classification → Cross-Entropy.** For $K$ classes the model outputs a probability vector
$\hat{y} = \text{softmax}(z)$, and the true label is a one-hot $y$. The negative
log-likelihood of the correct class is the **cross-entropy**:

$$L_{\text{CE}} = -\sum_{k=1}^{K} y_k \log \hat{y}_k.$$

This is also the KL divergence between the true and predicted label distributions (up to a
constant), which is why it's the natural "distance between distributions" loss.

**Regularization has a Bayesian reading too.** Adding $\lambda\lVert\theta\rVert_2^2$ (L2 /
"weight decay") is exactly placing a Gaussian *prior* on the weights and doing MAP
estimation instead of MLE; $\lambda\lVert\theta\rVert_1$ (L1) is a Laplace prior, whose
sharp peak at zero is why L1 drives weights *exactly* to zero and yields sparse models.
Regularization isn't a hack — it's a prior belief that simpler models generalize better.

## 3. The Bias–Variance Decomposition (Why Overfitting Is Mathematically Inevitable)

For squared loss, the expected test error at a point decomposes exactly. Let $\hat{f}$ be
the model learned from a random training set; then

$$\mathbb{E}\big[(y - \hat{f}(x))^2\big] = \underbrace{\big(\mathbb{E}[\hat{f}(x)] - f(x)\big)^2}_{\text{bias}^2} + \underbrace{\mathbb{E}\big[(\hat{f}(x) - \mathbb{E}[\hat{f}(x)])^2\big]}_{\text{variance}} + \underbrace{\sigma^2}_{\text{irreducible}}$$

Reading it: **bias** is error from wrong assumptions (too simple a model — underfitting);
**variance** is sensitivity to the particular training sample (too flexible — overfitting);
**irreducible** noise is the floor no model beats. Model complexity trades bias for
variance, and the classic U-shaped test-error curve is this trade playing out. (The modern
wrinkle — *double descent*, where hugely overparameterized nets improve again past the
interpolation point — is an active-research complication to this classical picture, worth
knowing exists.) This decomposition is *why* you need a validation set: to find the
complexity that minimizes the sum, not either term alone.

## 4. Linear Models, Done Properly

**Linear regression** predicts $\hat{y} = X\theta$ and minimizes
$\lVert X\theta - y\rVert^2$. Setting the gradient to zero,
$\nabla_\theta = 2X^\top(X\theta - y) = 0$, gives the **normal equations** in closed form:

$$\theta^* = (X^\top X)^{-1} X^\top y.$$

Geometrically, $X\theta^*$ is the orthogonal **projection** of $y$ onto the column space of
$X$ — the residual $y - X\theta^*$ is perpendicular to every feature. That's what "least
squares" means geometrically, and it's why $X^\top(\text{residual}) = 0$.

**Logistic regression** handles binary classification. It squashes a linear score through
the **sigmoid**

$$\sigma(z) = \frac{1}{1 + e^{-z}}, \qquad \hat{y} = \sigma(\theta^\top x),$$

interpreting $\hat{y}$ as $p(y=1\mid x)$; equivalently $\theta^\top x$ is the **log-odds**
$\log\frac{p}{1-p}$. Plugging into cross-entropy and differentiating yields a strikingly
clean gradient — identical in form to linear regression's:

$$\nabla_\theta L = X^\top(\hat{y} - y).$$

That "prediction minus target, times inputs" form is not a coincidence; it holds for the
whole family of generalized linear models with their canonical link, and it's the gradient
your from-scratch code will compute. For $K$ classes, sigmoid generalizes to **softmax**,
$\hat{y}_k = e^{z_k} / \sum_j e^{z_j}$, and the gradient of softmax-with-cross-entropy
collapses to the same beautiful $\hat{y} - y$.

## 5. Gradient Descent — the Universal Engine

Almost nothing above has a closed form once models get complex, so we descend. The update:

$$\theta \leftarrow \theta - \eta\,\nabla_\theta L(\theta)$$

**Why the negative gradient?** The gradient $\nabla_\theta L$ points in the direction of
steepest *increase*; a first-order Taylor expansion $L(\theta + \delta) \approx L(\theta) +
\nabla L^\top \delta$ is minimized (for a fixed small step) by moving opposite the gradient.
The **learning rate** $\eta$ sets the step: too small crawls; too large overshoots and
diverges (oscillating across the valley). This one hyperparameter causes more failed
training runs than any other.

**Batch vs stochastic vs mini-batch.** Full-batch uses all $n$ points per step (accurate
gradient, expensive). **Stochastic** GD uses one point (noisy but cheap, and the noise
helps escape shallow minima). **Mini-batch** (typically 32–512) is the practical sweet
spot — enough samples for a stable estimate, small enough to step often and to exploit
vectorized hardware. The noise in SGD is a *feature*: it's implicit regularization.

**Momentum** accumulates an exponentially-weighted average of past gradients to damp
oscillation and accelerate along consistent directions:

$$v \leftarrow \beta v + (1-\beta)\nabla_\theta L, \qquad \theta \leftarrow \theta - \eta v.$$

**Adam**, today's default, keeps two running moments — the mean $m$ and uncentered variance
$v$ of the gradient — and gives each parameter its own adaptive step:

$$m_t = \beta_1 m_{t-1} + (1-\beta_1)g_t, \quad v_t = \beta_2 v_{t-1} + (1-\beta_2)g_t^2,$$

then bias-corrects ($\hat{m}_t = m_t/(1-\beta_1^t)$, likewise $\hat v_t$) and updates
$\theta \leftarrow \theta - \eta\,\hat{m}_t / (\sqrt{\hat{v}_t} + \epsilon)$. The division by
$\sqrt{\hat v_t}$ shrinks steps for high-variance parameters — a per-weight learning rate.

**Convexity** is why this all works — or doesn't. For convex losses (linear/logistic
regression) any local minimum is global, so GD provably converges. Neural nets are
**non-convex**; we accept local minima and saddle points, and it turns out that in high
dimensions most critical points are saddles, not bad minima, and good-enough solutions are
abundant. That empirical fact is much of why deep learning works at all.

## 6. Neural Networks and Backpropagation — the Heart

A feedforward net stacks affine maps and nonlinearities. Layer $l$ computes

$$z^{(l)} = W^{(l)} a^{(l-1)} + b^{(l)}, \qquad a^{(l)} = \phi(z^{(l)}),$$

with $a^{(0)} = x$. The nonlinearity $\phi$ is essential: without it, a stack of linear
maps collapses to a single linear map (compose $W_2(W_1 x) = (W_2 W_1)x$) and you've gained
nothing. The **universal approximation theorem** says one hidden layer of sufficient width
can approximate any continuous function — existence, not a recipe; depth is what makes it
*efficient*.

**Backpropagation is just the chain rule, organized.** Define the error at layer $l$ as
$\delta^{(l)} = \partial L / \partial z^{(l)}$. The four equations of backprop:

$$\delta^{(L)} = \nabla_a L \odot \phi'(z^{(L)}) \qquad\text{(output layer)}$$

$$\delta^{(l)} = \big((W^{(l+1)})^\top \delta^{(l+1)}\big) \odot \phi'(z^{(l)}) \qquad\text{(propagate backward)}$$

$$\frac{\partial L}{\partial W^{(l)}} = \delta^{(l)} (a^{(l-1)})^\top, \qquad \frac{\partial L}{\partial b^{(l)}} = \delta^{(l)}$$

where $\odot$ is elementwise product. Read them as a story: compute the output error, push
it backward through the transposed weights (that's the chain rule threading through each
layer), gate it by the local slope $\phi'$, and read off each weight's gradient as
"downstream error × upstream activation." One forward pass caches the $a$'s and $z$'s; one
backward pass reuses them — total cost about twice a forward pass, which is the efficiency
that makes training feasible. **This is exactly what Nabla implements**, but generalized:
instead of hand-derived layer equations, it builds a graph of scalar operations and applies
the chain rule to each, so backprop emerges from the graph automatically (reverse-mode
autodiff).

**Activations and their derivatives** (the $\phi'$ above):

| $\phi$ | $\phi(z)$ | $\phi'(z)$ | Note |
|---|---|---|---|
| Sigmoid | $1/(1+e^{-z})$ | $\phi(1-\phi)$ | Saturates → vanishing gradients |
| Tanh | $\tanh z$ | $1 - \phi^2$ | Zero-centered sigmoid |
| ReLU | $\max(0, z)$ | $\mathbf{1}[z>0]$ | Cheap, non-saturating → won deep learning |

**Vanishing/exploding gradients:** the backward recursion multiplies by $\phi'$ and $W$ at
every layer. If those factors are $<1$ (sigmoid saturates near 0/1, where $\phi'\to 0$),
the gradient shrinks geometrically with depth and early layers stop learning; if $>1$, it
explodes. ReLU's derivative is exactly 1 on the active side — no shrinkage — which, with
careful initialization (He/Xavier, scaling variance to preserve signal magnitude across
layers) and normalization (batch/layer norm), is what made training deep nets practical.

## 7. Evaluating Models Honestly

- **The split.** Train on one set, tune hyperparameters on a **validation** set, and touch
  the **test** set once, at the end. Every time a decision is influenced by the test set, it
  leaks and stops estimating generalization. **k-fold cross-validation** rotates the
  validation fold to use data efficiently when it's scarce.
- **Classification metrics.** Accuracy lies on imbalanced data (99% accuracy is trivial
  if 99% of examples are one class). From the confusion matrix:
  $\text{precision} = \frac{TP}{TP+FP}$ ("of what I flagged, how much was right"),
  $\text{recall} = \frac{TP}{TP+FN}$ ("of what's actually positive, how much I caught"),
  and their harmonic mean $F_1 = \frac{2\,PR}{P+R}$. **ROC-AUC** measures ranking quality
  across all thresholds. Which matters depends on the cost of each error type — a cancer
  screen wants recall; a spam filter wants precision.
- **Calibration** — separate from accuracy — asks whether predicted probabilities are
  truthful (of things you called 70% likely, do 70% happen?). Often neglected, often
  what matters for decisions.

## 8. The Map Beyond Feedforward Nets

Lighter on math, enough to orient. Each architecture is an *inductive bias* — a structural
assumption that makes learning a certain data type efficient:

- **CNNs** replace full connectivity with a small **convolution** kernel slid across the
  input, sharing weights spatially. Two consequences: far fewer parameters, and
  **translation equivariance** (a feature is detected wherever it appears). The bias "nearby
  pixels relate, and location shouldn't matter" is why CNNs suit images.
- **RNNs/LSTMs** process sequences by carrying a hidden state $h_t = \phi(W_h h_{t-1} + W_x
  x_t)$. They struggle with long dependencies because backprop-through-time multiplies many
  Jacobians — the vanishing-gradient problem along the *time* axis. LSTMs add gated memory
  to mitigate it. Largely superseded by attention, but the sequence-modeling framing is
  foundational.

## 9. Attention and Transformers — What Modern "AI Models" Run On

Transformers power today's large language models, so they earn real math. The core operation
is **scaled dot-product attention**. Every token emits three vectors — a **query** $q$, a
**key** $k$, and a **value** $v$ (each a learned linear projection of the token's embedding).
A token gathers information by attending to others: it compares its query to every key,
softmaxes the scores into weights, and takes the weighted sum of values. Stacked into
matrices $Q, K, V$:

$$\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V$$

Each piece:
- $QK^\top$ is every query dotted with every key — an $n\times n$ matrix of similarity
  scores, "how relevant is token $j$ to token $i$."
- The **$\sqrt{d_k}$ scaling** matters: for $d_k$-dimensional vectors the dot products have
  variance $\propto d_k$, so without scaling the softmax inputs grow large, the softmax
  saturates, and gradients vanish. Dividing by $\sqrt{d_k}$ keeps the variance $\approx 1$.
- **softmax** turns each row of scores into attention weights summing to 1.
- Multiplying by $V$ takes the weighted average of value vectors — the actual information
  moved between tokens.

**Multi-head** attention runs several of these in parallel with different projections, so
the model attends to different relationships (syntax, coreference, position) at once, then
concatenates. Because attention is permutation-invariant, transformers add **positional
encodings** to inject order. A transformer block is attention + a per-token MLP + residual
connections + layer norm, stacked $N$ deep. The reason attention displaced RNNs: it relates
any two positions in *one* step (no vanishing gradient through time) and it's massively
parallelizable, which is what let models scale.

**LLMs** are transformers trained by **self-supervision** on next-token prediction:
minimize cross-entropy on "predict token $t+1$ from tokens $1..t$" over enormous text
corpora — no human labels needed, the data labels itself. The surprise of the last few
years is the **scaling hypothesis**: capabilities emerge predictably as parameters, data,
and compute grow together. "Training" learns the base model; **fine-tuning** adapts it to a
task; **in-context learning** (few-shot prompting) adapts it with *zero* weight updates,
purely from the prompt — an emergent behavior nobody explicitly trained for. When you build
on the Claude API (see the workspace's `claude-api` skill), this is the machinery underneath
the tokens you send.

## 10. The Engineering Reality Courses Skip

The math above is maybe 20% of a working ML project. The rest, and where the handbook's
engineering practices apply directly:

- **Data is the job.** Collection, cleaning, labeling, and — the silent killer —
  **leakage**: any way test-time-unavailable information sneaks into training (a feature
  computed using the future, a normalization fit on the whole dataset before splitting).
  Leakage produces gorgeous validation numbers and a model that fails in production. Suspect
  it whenever results look too good.
- **Distribution shift.** The i.i.d. assumption of §1 is a convenient lie; production data
  drifts from training data, and models silently rot. Monitoring for it is an observability
  problem ([debugging module §6](09-debugging-and-observability.md)).
- **Reproducibility & experiment tracking.** Seeds, pinned data versions, logged
  hyperparameters and metrics — an ML experiment is a scientific one, and the
  version-control and documentation discipline from the handbook applies with extra force
  because results depend on data *and* code *and* randomness.
- **The train/serve skew.** The code that featurizes data in training and the code that does
  it in production must agree exactly; when they drift, the model sees inputs it was never
  trained on. This is the ML version of an API-contract violation.

## 11. Lab Track

Anchored on [Nabla](../project-9-nabla/), the from-scratch autodiff engine, plus reading.

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Read `nabla/value.py` and its `backward()` with §6 open — match each operator's local gradient to the chain rule | Nabla | Backprop as code, not a formula |
| 2 | Run the training demo; watch cross-entropy fall and accuracy rise on a real toy dataset; change the learning rate to see §5's "too big/too small" live | Nabla | Gradient descent, felt |
| 3 | Read the **gradient-checking** tests — autograd gradients verified against finite differences $\frac{L(\theta+\epsilon)-L(\theta-\epsilon)}{2\epsilon}$ | Nabla | The calculus, empirically confirmed — and how you'd ever trust an autodiff engine |
| 4 | Sprint 2: add an activation or the Adam optimizer from §5's equations; add L2 regularization and watch the variance drop | Nabla | Turning §5 math into working code |
| 5 | Derive the softmax-cross-entropy gradient by hand and confirm it equals $\hat{y}-y$ (§4); then find where Nabla computes it | Nabla + paper | Closing the loop between derivation and implementation |

**Research prompts:** the reparameterization trick (VAEs); why batch norm works (still
debated); attention's $O(n^2)$ cost and the efficient-attention line of work (FlashAttention,
linear attention); the lottery-ticket hypothesis; double descent and why classical
bias-variance isn't the whole story; RLHF/DPO for aligning LLMs; diffusion models as
score-matching; the bitter lesson (Sutton) — why general methods that scale beat
hand-engineered cleverness.
