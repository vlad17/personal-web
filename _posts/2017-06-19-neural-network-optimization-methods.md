---
layout: post
title:  "Neural Network Optimization Methods"
date:   2017-06-19
categories: machine-learning optimization deep-learning
meta_keywords: optimization, neural networks, stochastic gradient descent
---

# Neural Network Optimization Methods

The goal of this post and its related sub-posts is to explore at a high level how the theoretical guarantees of the various optimization methods interact with non-convex problems in practice, where we don't really know Lipschitz constants, the validity of the assumptions that these methods make, or appropriate hyperparameters. Obviously, a detailed treatment would require delving into intricacies of cutting-edge research. That's not the point of this post, which just seeks to offer a theoretical survey.

I should also caution the reader that I'm not drawing on any of my own experience when discussing "practical" aspects of neural network (NN) optimization, but rather [Dr. Goodfellow's](http://www.deeplearningbook.org/). For the most part, I'll be summarizing sections 8.5 and 8.6 of the [optimization chapter](http://www.deeplearningbook.org/contents/optimization.html) in that book, but I'll throw in some relevant background and research, too. Further, one departure from practicality that I'll be making for simplicity is not considering parallelism. All mentioned analyses assume sequential execution, and may not have obvious parallel versions. Even if they do, most bets are off.

In part, I'll also try to address exactly what theoretical guarantees we do have in a NN setting. Lots of work has been done for convex and adversarial online convex optimization, and most NNs are optimized by just throwing such a method at training. Luckily, a lot of very recent work, as of this posting, has addressed exactly what happens in this situation.

## Setting

A NN is a real-valued circuit \\(\hat{y}_\bsth\\) of computationally efficient, differentiable, and Lipschitz functions parameterized by \\(\bsth\\). This network is trained to minimize a loss, \\(J(\bsth)\\), based on empirical risk minimization (ERM). This is the hard part, computationally, for training NNs. We are given a set of supervised examples, pairs \\(\vx^{(i)},y^{(i)}\\) for \\(i\in[n]\\). Under the assumption that these pairs are coming from some fixed, unknown distribution, some learning can be done by ERM relative to a loss \\(\ell\\) on our training set, which amounts to the following:
\\[
\argmin\_\bsth J(\bsth) = \argmin\_\bsth \frac{1}{n}\sum\_{i=1}^m\ell(\hat{y}\_\bsth(\vx^{(i)}), y^{(i)})+\Omega(\bsth)
\\]
Above, \\(\Omega\\) is a regularization term (added to restrict the hypothesis class). Its purpose is for generalization. Typically, \\(\Omega\\) is of the form of an \\(L^2\\) or \\(L^1\\) norm. In other cases, it has a more complicated implicit form such as the case when we perform model averaging through dropout or weight regularization through early stopping (regularization may also be some kind of smoothing, like gradient clipping). In any case, we will assume that there exist some general strategies for reducing problems with nonzero \\(\Omega\\) to those where it is zero (see, for example, analysis and references in [Krogh and Hertz 1991](https://papers.nips.cc/paper/563-a-simple-weight-decay-can-improve-generalization), [Beck and Teboulle 2009](http://epubs.siam.org/doi/abs/10.1137/080716542), [Allen-Zhu 2016](https://arxiv.org/abs/1603.05953)). The presence of regularization in general is nuanced, and its application requires deeper analyses for minimization methods, but we will skirt those concerns when discussing practical behavior for the time being.

An initial source of confusion about the above machine learning notation is the reuse of variable names in the optimization literature, where instead our parameters \\(\bsth\\) are points \\(\vx\\) and our training point errors \\(\ell(\hat{y}\_\bsth(\vx^{(i)}), y^{(i)})\\) are replaced with opaque Lipschitz, differentiable costs \\(f_i(\vx)\\). We now summarize our general task of (unconstrained) NN optimization of our nonconvex composite (regularized) cost function \\(f:\R^d\rightarrow \R\\):
\\[
\argmin\_\vx f(\vx) = \argmin\_\vx \frac{1}{n}\sum\_{i=1}^nf\_i(\vx)+\Omega(\vx)
\\]

In a lot of literature inspiring these algorithms, it's important to keep straight in one's head the various types of minimization problems that are being solved, and whether they're making incompatible assumptions with the NN environment.

* Many algorithms are inspired by the general convex \\(f\\) case. NN losses are usually not convex.
* Sometimes, full gradients \\(\nabla f\\) are assumed. A full gradient is intractable for NNs, as it requires going through the entire \\(m\\)-sized dataset. We are looking for stochastic approximations to the gradient \\(\E\ha{\tilde{\nabla} f}=\nabla f\\).
* Some algorithms assume \\(\Omega = 0\\), but that's not usually the case.

## Theoretical Convergence

We'll be looking at gradient descent (GD) optimization algorithms, which assume an initial point \\(\vx\_0\\) and move in nearby directions to reduce the cost. As such, basically all asymptotic rates contain a hidden constant multiplicative term \\(f(\vx\_0) - \inf f\\).

### Problem Specification

Before discussing speed, it's important to know what constitutes a solution. Globally minimizing a possibly non-convex function such as deep NN is NP-hard. Even finding an approximate local minimum of just a quartic multivariate polynomial or showing its convexity is NP-hard ([Ahmadi et al 2010](https://arxiv.org/abs/1012.1908)).

What we do, in theory, at least, is instead merely find **approximate critical points**; i.e., a typical non-convex optimization algorithm would return a point \\(\vx\_{\*}\\) that satisfies \\(\norm{\nabla f(\vx\_\*)}\le \epsilon\\). This is an **incredibly weak** requirement: for NNs, there are significantly more saddle points than local minima, and they have high cost. Luckily, local minima actually concentrate around the global minimum cost for NNs, as opposed to saddles, so recent cutting-edge methods that find approximate local minima are worth keeping in mind. An approximate local minimum \\(\vx\_*\\) has a neighborhood such that any \\(\vx\\) in that neighborhood will have \\(f(\vx)-f(\vx\_\*)\le \epsilon\\). [See extended discussion here.](https://github.com/vlad17/ml-notes/blob/master/deep-learning/optimization.pdf)

We'll assume that \\(f\\) is differentiable and Lipschitz. Even though ReLU activations and \\(L^1\\) regularization may technically invalidate the differentiability, these functions have well-defined **subgradient** that respect [GD properties that we care about](http://web.stanford.edu/class/msande318/notes/notes-first-order-nonsmooth.pdf). Certain algorithms further might assume \\(f\in\mathcal{C}^2\\) and that the Hessian is operator-norm Lipschitz or bounded.

There are two main runtime costs. The first is the desired degree of accuracy, \\(\epsilon\\). The second is due to the dimensionality of our input \\(d\\). Ignoring representation issues, thanks to the circuit structure of \\(f\\), we evaluate for any \\(i\in[n]\\) and \\(\vv\in\R^d\\) all of \\(f\_i(\vx), \nabla f\_i(\vx), {\\nabla^2 f_i(\vx)} \vv\\) in \\(O(d)\\) time. Finally, since gradients of \\(f\_i\\) approximate gradients of \\(f\\) only *in expectation*, reported worst-case runtimes are usually worst-case runtimes such that we *expect* to arrive at an approximate stationary point (expectation taken over the random uniform selection of \\(i\\) in SGD).

### Fundamental Lower Bounds

First, unless \\(\ptime = \nptime\\), we expect runtime to be at least \\(\Omega\pa{\log\frac{d}{\epsilon}}\\) due to the aforementioned hardness results.
fluctuation.

Less obviously, convex optimization lower bounds for smooth functions imply that any first-order non-convex algorithm requires at least \\(\Omega(1/\epsilon)\\) gradient steps ([Bubek 2014](https://arxiv.org/abs/1405.4980), see also notes [here](http://www.stat.cmu.edu/~larry/=sml/optrates.pdf) and [here](http://www.cs.cmu.edu/~suvrit/teach/aaditya_lect23.pdf)). Note that this is nowhere near polynomial time in the bit size of \\(\epsilon\\)!

See also [Wolpert and Macready 1997](http://ieeexplore.ieee.org/document/585893/), [Blum and Rivest 1988](https://papers.nips.cc/paper/125-training-a-3-node-neural-network-is-np-complete), and a recent re-visiting of the topic in [Livni et al 2014](https://arxiv.org/abs/1410.1141). In other words, general non-convex optimization time lower bounds are too broad to apply usefully to NN, but specific approaches to fixed architectures may be appropriate.

### Limitations of Theoretical Descriptions

There are a couple of limitations in using asymptotic, theoretical descriptions of convergence rates to analyze these algorithms.

First, the \\(\epsilon\\) in \\(\epsilon\\)-approximate critical points above is merely a small piece in the overall generalization error that the NN will experience. As explained in [Bousquet and Bottou 2007](https://papers.nips.cc/paper/3323-the-tradeoffs-of-large-scale-learning) ([extended version](http://leon.bottou.org/papers/bottou-bousquet-2011)), the generalization error is broken into approximation (how accurate the entire function class of neural networks for a fixed architecture is in representing the true function we're learning), estimation (how far we are from a global optimum among our hypothesis class of functions), and optimization error (our convergence tolerance). As cautioned in the aforementioned paper, the tradeoff between the aforementioned errors implies that even improvements in optimization convergence rate, like the use of full GD instead of stochastic GD (SGD) may not be helpful if they increase other errors in hidden ways.

Second, early stopping might prevent prevent convergence altogether---as mentioned in Goodfellow's book, gradient norms can increase while training error decreases. It's unclear whether we can fold in early stopping as an implicit term in \\(\Omega\\) and claim that we're reaching a critical point in this virtual cost function. 

The fact that that theoretical lower bound rates are not relevant for NN training time (compared to what we see in practice) shows that there is a wide gap between general non-convex and smooth approximate local minimum finding and the same problem for NNs. 

## Existing Algorithms

In the below two linked blog posts, I will review the high-level details existing algorithms for NN non-convex optimization. Most of these are methods that have been developed for the composite *convex* smooth optimization problem, so they may not even have any theoretical guarantees for the \\(\epsilon\\)-approximate critical point or local min problem. It turns out that indeed we find a general dichotomy between these GD algorithms:

* Algorithms which are practically available, e.g., [TensorFlow's first order methods](https://www.tensorflow.org/api_guides/python/train), but were initially developed for convex problems, and whose non-convex interpretations are usually only approximate critical point finders
* Algorithms which are (as of June 2017) cutting-edge research and not widely available, yet have been designed for finding local minima efficiently in non-convex settings. Nonetheless, they're still useful to mention since the respective paper implementations might be available and it may be worthwhile to manually implement the optimization, too.

This list of existing algorithms is going to be a bit redundant with the review [Ruder 2016](https://arxiv.org/abs/1609.04747), but my intention is to be a bit more comprehensive and rigorous but less didactic in terms of update rules covered.

In general, all these rules have the format \\(\vx\_{t+1}=\vx\_t-\eta\_t\vg\_t\\) where \\(\eta\_t\\) is a learning rate and \\(\vg\_t\\) is the gradient descent direction, both making a small local improvement at the \\(t\\)-th discrete time. Theoretical analysis won't be presented, but guarantees, assumptions, intuition, and update rules will be described. Proofs will be linked.

* [First order methods]({{ site.baseurl }}{% post_url 2017-06-20-nonconvex-first-order-methods %})

Technically, most neural networks don't have smoothness or even differentiability everywhere. While in reality those issues don't seem to surface in practice, it turns out we can [still make some strong statements](https://arxiv.org/abs/1804.07795) about first-order optimization methods.
