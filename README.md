# Ebisu: intelligent quiz scheduling

## Important links

- [Literate document](https://fasiha.github.io/ebisu/)
- [GitHub repo](https://github.com/fasiha/ebisu)
- [IPython Notebook crash course](https://github.com/fasiha/ebisu/blob/gh-pages/EbisuHowto.ipynb)
- [PyPI package](https://pypi.python.org/pypi/ebisu/)
- [Contact](https://fasiha.github.io/#contact)

### Table of contents

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Important links](#important-links)
	- [Table of contents](#table-of-contents)
- [Introduction](#introduction)
- [Quickstart](#quickstart)
- [How it works](#how-it-works)
- [The math](#the-math)
	- [Bernoulli quizzes](#bernoulli-quizzes)
	- [Moving Beta distributions through time](#moving-beta-distributions-through-time)
	- [Mean and variance of the recall probability right now](#mean-and-variance-of-the-recall-probability-right-now)
	- [Choice of initial model parameters](#choice-of-initial-model-parameters)
	- [Updating the posterior with quiz results](#updating-the-posterior-with-quiz-results)
- [Source code](#source-code)
	- [Core library](#core-library)
	- [Miscellaneous functions](#miscellaneous-functions)
	- [Test code](#test-code)
- [Demo codes](#demo-codes)
	- [Visualizing half-lives](#visualizing-half-lives)
	- [Why we work with random variables](#why-we-work-with-random-variables)
- [Requirements for building all aspects of this repo](#requirements-for-building-all-aspects-of-this-repo)
- [Acknowledgements](#acknowledgements)

<!-- /TOC -->

## Introduction

Consider a student memorizing a set of facts.

- Which facts need reviewing?
- How does the student’s performance on a review change the fact’s future review schedule?

Ebisu is a public-domain library that answers these two questions. It is intended to be used by software developers writing quiz apps, and provides a simple API to deal with these two aspects of scheduling quizzes:
- `predictRecall` gives the current recall probability for a given fact.
- `updateRecall` adjusts the belief about future recall probability given a quiz result.

Behind these two simple functions, Ebisu is using a simple yet powerful model of forgetting, a model that is founded on Bayesian statistics and exponential forgetting.

With this system, quiz applications can move away from “daily review piles” caused by less flexible scheduling algorithms. For instance, a student might have only five minutes to study today; an app using Ebisu can ensure that only the facts most in danger of being forgotten are reviewed.

Ebisu also enables apps to provide an infinite stream of quizzes for students who are cramming. Thus, Ebisu intelligently handles over-reviewing as well as under-reviewing.

This document is a literate source: it contains a detailed mathematical description of the underlying algorithm as well as source code for a Python implementation (requires Scipy and Numpy). Separate implementations in [JavaScript (Ebisu.js)](https://fasiha.github.io/ebisu.js/) and [Java (ebisu-java)](https://github.com/fasiha/ebisu-java) exist.

The next section is a [Quickstart](#quickstart) guide to setup and usage. See this if you know you want to use Ebisu in your app.

Then in the [How It Works](#how-it-works) section, I contrast Ebisu to other scheduling algorithms and describe, non-technically, why you should use it.

Then there’s a long [Math](#the-math) section that details Ebisu’s algorithm mathematically. If you like Beta-distributed random variables, conjugate priors, and marginalization, this is for you. You’ll also find the key formulas that implement `predictRecall` and `updateRecall` here.

> Nerdy details in a nutshell: Ebisu begins by positing a [Beta prior](https://en.wikipedia.org/wiki/Beta_distribution) on recall probabilities. As time passes, the recall probability decays exponentially, and Ebisu handles that nonlinearity exactly and analytically—it requires only a few [gamma function](http://mathworld.wolfram.com/GammaFunction.html) evaluations to predict the current recall probability. Next, a *quiz* is modeled as a [Bernoulli trial](https://en.wikipedia.org/wiki/Bernoulli_distribution) whose underlying probability prior is this non-conjugate nonlinearly-transformed Beta. Ebisu approximates the true non-standard posterior with a new Beta distribution by matching its mean and variance. This mean and variance are analytically tractable, and again require a few evaluations of the gamma function.

Finally, the [Source Code](#source-code) section presents the literate source of the library, including several tests to validate the math.

## Quickstart

**Install** `pip install ebisu` (both Python3 and Python2 ok 🤠).

**Data model** For each fact in your quiz app, you store a model representing a prior distribution. This is a 3-tuple: `(alpha, beta, t)` and you can create a default model for all newly learned facts with `ebisu.defaultModel`. (As detailed in the [Choice of initial model parameters](#choice-of-initial-model-parameters) section, `alpha` and `beta` define a Beta distribution on this fact’s recall probability `t` time units after it’s most recent review.)

**Predict a fact’s current recall probability** `ebisu.predictRecall(prior: tuple, tnow: float) -> float` where `prior` is this fact’s model, `tnow` is the current time elapsed since this fact’s most recent review, and the returned value is a probability between 0 and 1.

**Update a fact’s model with quiz results** `ebisu.updateRecall(prior: tuple, result: bool, tnow: float) -> tuple` where `prior` and `tnow` are as above, and where `result` is true if the student successfully answered the quiz, false otherwise. The returned value is this fact’s new prior model—the old one can be discarded.

**IPython Notebook crash course** For a conversational introduction to the API in the context of a mocked quiz app, see this [IPython Notebook crash course](https://github.com/fasiha/ebisu/blob/gh-pages/EbisuHowto.ipynb).

**Further information** [Module docstrings](https://github.com/fasiha/ebisu/blob/gh-pages/doc/doc.md) in a pinch but full details plus literate source below, under [Source code](#source-code).

**Alternative implementations** [Ebisu.js](https://fasiha.github.io/ebisu.js/) is a JavaScript port for browser and Node.js. [ebisu-java](https://github.com/fasiha/ebisu-java) is for Java and JVM languages.

## How it works

There are many scheduling schemes, e.g.,

- [Anki](https://apps.ankiweb.net/), an open-source Python flashcard app (and a closed-source mobile app),
- the [SuperMemo](https://www.supermemo.com/help/smalg.htm) family of algorithms ([Anki’s](https://apps.ankiweb.net/docs/manual.html#what-algorithm) is a derivative of SM-2),
- [Memrise.com](https://www.memrise.com), a closed-source webapp,
- [Duolingo](https://www.duolingo.com/) has published a [blog entry](http://making.duolingo.com/how-we-learn-how-you-learn) and a [conference paper/code repo](https://github.com/duolingo/halflife-regression) on their half-life regression technique,
- the Leitner and Pimsleur spacing schemes (also discussed in some length in Duolingo’s paper).
- Also worth noting is Michael Mozer’s team’s Bayesian multiscale models, e.g., [Mozer, Pashler, Cepeda, Lindsey, and Vul](http://www.cs.colorado.edu/~mozer/Research/Selected%20Publications/reprints/MozerPashlerCepedaLindseyVul2009.pdf)’s 2009 <cite>NIPS</cite> paper and subsequent work.

Many of these are inspired by Hermann Ebbinghaus’ discovery of the [exponential forgetting curve](https://en.wikipedia.org/w/index.php?title=Forgetting_curve&oldid=766120598#History), published in 1885, when he was thirty-five. He [memorized random](https://en.wikipedia.org/w/index.php?title=Hermann_Ebbinghaus&oldid=773908952#Research_on_memory) consonant–vowel–consonant trigrams (‘PED’, e.g.) and found, among other things, that his recall decayed exponentially with some time-constant.

Anki and SuperMemo use carefully-tuned mechanical rules to schedule a fact’s future review immediately after its current review. The rules can get complicated—I wrote a little [field guide](https://gist.github.com/fasiha/31ce46c36371ff57fdbc1254af424174) to Anki’s, with links to the source code—since they are optimized to minimize daily review time while maximizing retention. However, because each fact has simply a date of next review, these algorithms do not gracefully accommodate over- or under-reviewing. Even when used as prescribed, they can schedule many facts for review on one day but few on others. (I must note that all three of these issues—over-reviewing (cramming), under-reviewing, and lumpy reviews—have well-supported solutions in Anki by tweaking the rules and third-party plugins.)

Duolingo’s half-life regression explicitly models the probability of you recalling a fact as \\(2^{-Δ/h}\\), where Δ is the time since your last review and \\(h\\) is a *half-life*. In this model, your chances of passing a quiz after \\(h\\) days is 50%, which drops to 25% after \\(2 h\\) days. They estimate this half-life by combining your past performance and fact metadata in a large-scale machine learning technique called half-life regression (a variant of logistic regression or beta regression, more tuned to this forgetting curve). With each fact associated with a half-life, they can predict the likelihood of forgetting a fact if a quiz was given right now. The results of that quiz (for whichever fact was chosen to review) are used to update that fact’s half-life by re-running the machine learning process with the results from the latest quizzes.

The Mozer group’s algorithms also fit a hierarchical Bayesian model that links quiz performance to memory, taking into account inter-fact and inter-student variability, but the training step is again computationally-intensive.

Like Duolingo and Mozer’s approaches, Ebisu explicitly tracks the exponential forgetting curve to provide a list of facts sorted by most to least likely to be forgotten. However, Ebisu formulates the problem very differently—while memory is understood to decay exponentially, Ebisu posits a *probability distribution* on the half-life and uses quiz results to update its beliefs in a fully Bayesian way. These updates, while a bit more computationally-burdensome than Anki’s scheduler, are much lighter-weight than Duolingo’s industrial-strength approach.

This gives small quiz apps the same intelligent scheduling as Duolingo’s approach—real-time recall probabilities for any fact—but with immediate incorporation of quiz results, even on mobile apps.

To appreciate this further, consider this example. Imagine a fact with half-life of a week: after a week we expect the recall probability to drop to 50%. However, Ebisu can entertain an infinite range of beliefs about this recall probability: it can be very uncertain that it’ll be 50% (the “α=β=3” model below), or it can be very confident in that prediction (“α=β=12” case):

![figures/models.png](figures/models.png)

Under either of these models of recall probability, we can ask Ebisu what the expected half-life is after the student is quizzed on this fact a day, a week, or a month after their last review, and whether they passed or failed the quiz:

![figures/halflife.png](figures/halflife.png)

If the student correctly answers the quiz, Ebisu expects the new half-life to be greater than a week. If the student answers correctly after just a day, the half-life rises a little bit, since we expected the student to remember this fact that soon after reviewing it. If the student surprises us by *failing* the quiz just a day after they last reviewed it, the projected half-life drops. The more tentative “α=β=3” model aggressively adjusts the half-life, while the more assured “α=β=12” model is more conservative in its update. (Each fact has an α and β associated with it and I explain what they mean mathematically in the next section. Also, the code for these two charts is [below](#demo-codes).)

Similarly, if the student fails the quiz after a whole month of not reviewing it, this isn’t a surprise—the half-life drops a bit from the initial half-life of a week. If she does surprise us, passing the quiz after a month of not studying it, then Ebisu boosts its expectated half-life—by a lot for the “α=β=3” model, less for the “α=β=12” one.

> Currently, Ebisu treats each fact as independent, very much like Ebbinghaus’ nonsense syllables: it does not understand how facts are related the way Duolingo can with its sentences. However, Ebisu can be used in combination with other techniques to accommodate extra information about relationships between facts.

## The math

### Bernoulli quizzes

Let’s begin with a quiz. One way or another, we’ve picked a fact to quiz the student on, \\(t\\) days (the units are arbitrary since \\(t\\) can be any positive real number) after her last quiz on it, or since she learned it for the first time.

We’ll model the results of the quiz as a Bernoulli experiment, \\(x_t ∼ Bernoulli(p)\\); \\(x_t\\) can be either 1 (success) with probability \\(p_t\\), or 0 (fail) with probability \\(1-p_t\\). Let’s think about \\(p_t\\) as the recall probability at time \\(t\\)—then \\(x_t\\) is a coin flip, with a \\(p_t\\)-weighted coin.

The [Beta distribution](https://en.wikipedia.org/wiki/Beta_distribution) happens to be the [conjugate prior](https://en.wikipedia.org/wiki/Conjugate_prior) for the Bernoulli distribution. So if our *a priori* belief about \\(p_t\\) follow a Beta distribution, that is, if
\\[p_t ∼ Beta(α_t, β_t)\\]
for specific \\(α_t\\) and \\(β_t\\), then observing the quiz result updates our belief about the recall probability to be:
\\[p_t | x_t ∼ Beta(α_t + x_t, β_t + 1 - x_t).\\]

> **Aside 1** Notice that since \\(x_t\\) is either 1 or 0, the updated parameters \\((α + x_t, β + 1 - x_t)\\) are \\((α + 1, β)\\) when the student correctly answered the quiz, and \\((α, β + 1)\\) when she answered incorrectly.
>
> **Aside 2** Even if you’re familiar with Bayesian statistics, if you’ve never worked with priors on probabilities, the meta-ness here might confuse you. What the above means is that, before we flipped our \\(p_t\\)-weighted coin (before we administered the quiz), we had a specific probability distribution representing the coin’s weighting \\(p_t\\), *not* just a scalar number. After we observed the result of the coin flip, we updated our belief about the coin’s weighting—it *still* makes total sense to talk about the probability of something happening after it happens. Said another way, since we’re being Bayesian, something actually happening doesn’t preclude us from maintaining beliefs about what *could* have happened.

This is totally ordinary, bread-and-butter Bayesian statistics. However, the major complication arises when the experiment took place not at time \\(t\\) but \\(t_2\\): we had a Beta prior on \\(p_t\\) (probability of  recall at time \\(t\\)) but the test is administered at some other time \\(t_2\\).

How can we update our beliefs about the recall probability at time \\(t\\) to another time \\(t_2\\), either earlier or later than \\(t\\)?

### Moving Beta distributions through time

Our old friend Ebbinghaus comes to our rescue. According to the exponentially-decaying forgetting curve, the probability of recall at time \\(t\\) is
\\[p_t = 2^{-t/h},\\]
for some notional half-life \\(h\\). Let \\(t_2 = δ·t\\). Then,
\\[p_{t_2} = p_{δ t} = 2^{-δt/h} = (2^{-t/h})^δ = (p_t)^δ.\\]
That is, to fast-forward or rewind \\(p_t\\) to time \\(t_2\\), we raise it to the \\(δ = t_2 / t\\) power.

Unfortunately, a Beta-distributed \\(p_t\\) becomes *non*-Beta-distributed when raised to any positive power \\(δ\\). For a quiz with recall probability given by \\(p_t ∼ Beta(12, 12)\\) for \\(t\\) one week after the last review (the middle histogram below), \\(δ > 1\\) shifts the density to the left (lower recall probability) while \\(δ < 1\\) does the opposite. Below shows the histogram of recall probability at the original half-life of seven days compared to that after two days (\\(δ = 0.3\\)) and three weeks (\\(δ  = 3\\)).
![figures/pidelta.png](figures/pidelta.png)

We could approximate this \\(δ\\) with a Beta random variable, but especially when over- or under-reviewing, the closest Beta fit is very poor. So let’s derive analytically the probability density function (PDF) for \\(p_t^δ\\). Recall the conventional way to obtain the density of a [nonlinearly-transformed random variable](https://en.wikipedia.org/w/index.php?title=Random_variable&oldid=771423505#Functions_of_random_variables): let \\(x=p_t\\) and \\(y = g(x) = x^δ\\) be the forward transform, so \\(g^{-1}(y) = y^{1/δ}\\) is its inverse. Then, with \\(P_X(x) = Beta(x; α,β)\\),
\\[P_{Y}(y) = P_{X}(g^{-1}(y)) · \frac{∂}{∂y} g^{-1}(y),\\]
and this after some Wolfram Alpha and hand-manipulation becomes
\\[P_{Y}(y) = y^{(α-δ)/δ} · (1-y^{1/δ})^{β-1} / (δ · B(α, β)),\\]
where \\(B(α, β) = Γ(α) · Γ(β) / Γ(α + β)\\) is [beta function](https://en.wikipedia.org/wiki/Beta_function), also the normalizing denominator in the Beta density (confusing, sorry), and \\(Γ(·)\\) is the [gamma function](https://en.wikipedia.org/wiki/Gamma_function), a generalization of factorial.

> To check this, type in `y^((a-1)/d) * (1 - y^(1/d))^(b-1) / Beta[a,b] * D[y^(1/d), y]` at [Wolfram Alpha](https://www.wolframalpha.com).

Replacing the \\(X\\)’s and \\(Y\\)’s with our usual variables, we have the probability density for \\(p_{t_2} = p_t^δ\\) in terms of the original density for \\(p_t\\):
\\[P(p_t^δ) = \frac{p^{(α - δ)/δ} · (1-p^{1/δ})^{β-1}}{δ · B(α, β)}.\\]

[Robert Kern noticed](https://github.com/fasiha/ebisu/issues/5) that this is a [generalized Beta of the first kind](https://en.wikipedia.org/w/index.php?title=Generalized_beta_distribution&oldid=889147668#Generalized_beta_of_first_kind_(GB1)), or GB1, random variable:
\\[p_t^δ ∼ GB1(p; a=1/δ, b=1, p=α; q=β)\\]
When \\(δ=1\\), that is, at exactly the half-life, recall probability is simply the initial Beta we started with.


We will use the density of \\(p_t^δ\\) to reach our two most important goals:
- what’s the recall probability of a given fact right now?, and
- how do I update my estimate of that recall probability given quiz results?

### Recall probability right now

Let’s see how to get the recall probability right now. Recall that we started out with a prior on the recall probabilities \\(t\\) days after the last review, \\(p_t ∼ Beta(α, β)\\). Letting \\(δ = t_{now} / t\\), where \\(t_{now}\\) is the time currently elapsed since the last review, we saw above that \\(p_t^δ\\) is GB1-distributed. [Wikipedia](https://en.wikipedia.org/w/index.php?title=Generalized_beta_distribution&oldid=889147668#Generalized_beta_of_first_kind_(GB1)) kindly gives us an expression for the expected recall probability right now, in terms of the Beta function, which we may as well simplify to Gamma function evaluations:
\begin{align}
E[p_t^δ] \\&= \frac{Γ(α + β)}{Γ(α)} · \frac{Γ(α + δ)}{Γ(α + β + δ)} \\\\
         \\&= γ_1 / γ_0,
\end{align}
where \\(γ_n = Γ(α + n·δ) / Γ(α+β+n·δ)\\).

A quiz app can calculate the average current recall probability for each fact using this formula, and thus find the fact most at risk of being forgotten.

### Choice of initial model parameters
Mentioning a quiz app reminds me—you may be wondering how to pick the prior triple \\([α, β, t]\\) initially, for example when the student has first learned a fact.

For \\(t\\)—I propose setting \\(t\\) equal to your best guess of the fact’s half-life. In Memrise, the first quiz occurs four hours after first learning a fact; in Anki, it’s a day after. To mimic these, set \\(t\\) to four hours or a day, respectively.

Then, set \\(α = β ≥ 2\\): the \\(α = β\\) part will center the Beta distribution for \\(p_t\\) at 0.5 (making \\(t\\) an actual half-life). A higher value for \\(α = β\\) encodes *higher* confidence in the expected half-life \\(t\\), which in turn makes the model *less* sensitive to quiz results (as we’ll show in the next section). A good default is \\(α = β = 3\\), which lets the algorithm aggressively change the half-life in response to quiz results.

If a student indicates they’re familiar with a flashcard already, quiz apps should increase the initial half-life \\(t\\), or in the opposite case, lower the half-life for flashcards that a student is very tentative about. This should be the default way of handling easy or hard cards. Varying \\(α = β\\) for different flashcards does a different thing: it makes the algorithm more or less confident that that half-life is correct. We are still experimenting with whether varying \\(α = β\\) for initial flashcards makes sense.

> Recall the traditional explanation of \\(α\\) and \\(β\\) are the number of successes and failures, respectively, that have been observed by flipping a weighted coin—or in our application, the number of successful versus unsuccessful quiz results for a sequence of quizzes on the same fact \\(t\\) days apart.
>
> Also recall that [above](#how-it-works) we visually contrasted the \\(α = β = 3\\) model (very sensitive and responsive) to the much more conservative \\(α = β = 12\\) model.

Now, let us turn to the final piece of the math, how to update our prior on a fact’s recall probability when a quiz result arrives.

### Updating the posterior with quiz results

One option could be this: since we have analytical expressions for the mean and variance of the prior on \\(p_t^δ\\), convert these to the [closest Beta distribution](https://en.wikipedia.org/w/index.php?title=Beta_distribution&oldid=774237683#Two_unknown_parameters) and straightforwardly update with the Bernoulli likelihood as mentioned [above](#bernoulli-quizzes). However, we can do much better.

By application of Bayes rule, the posterior is
\\[Posterior(p|x) = \frac{Prior(p) · Lik(x|p)}{\int_0^1 Prior(p) · Lik(x|p) \\, dp}.\\]
Here, “prior” refers to the GB1 density \\(P(p_t^δ)\\) derived above. \\(Lik\\) is the Bernoulli likelihood: \\(Lik(x|p) = p\\) when \\(x=1\\) and \\(1-p\\) when \\(x=0\\). The denominator is the marginal probability of the observation \\(x\\). (In the above, all recall probabilities \\(p\\) and quiz results \\(x\\) are at the same \\(t_2 = t · δ\\), but we’ll add time subscripts again below.)

We’ll break up the posterior into two cases, depending on whether the quiz is successful \\(x=1\\), or unsuccessful \\(x=0\\).

For the \\(x=1\\) case (successful quiz), the posterior is actually conjugate, and felicitously remains a GB1 random variable:
\\[(p_{t_2} | x_{t_2} = 1) ∼ GB1(p; a=1/δ, b=1, p=α+δ; q=β).\\]
To highlight the difference between this and the prior, note that the posterior has parameter \\(p=α+δ\\), whereas the prior in the previous section had \\(p=α\\). This makes it effortless to express the distribution of the recall probability at the original half-life \\(t\\), instead of at the quiz time \\(t_2\\) which might be very small or large (over- or under-review):
\\[(p_t | x_{t_2} = 1) ∼ Beta(α+δ, β)\\].

Now consider the \\(x=0\\) case, for an unsuccessful quiz. The posterior in this case is not conjugate, but we can analytically derive its moments (its mean, its non-central variance, etc.):
\\[E\\left[(p_{t_2} | x_{t_2} = 0\\right)^n] = \frac{γ_{n+1} - γ_n}{γ_1 - γ_0}.\\]
Recall that \\(γ_n = Γ(α + n·δ) / Γ(α+β+n·δ)\\), for the \\(α, β, δ=t_2/t\\) from the prior.

With this you *could* [fit](https://en.wikipedia.org/w/index.php?title=Beta_distribution&oldid=774237683#Two_unknown_parameters) a new Beta using the mean \\(E\\left[p_{t_2} | x_{t_2} = 0\\right]\\) and variance \\(E\\left[(p_{t_2} | x_{t_2} = 0\\right)^2] - E\\left[p_{t_2} | x_{t_2} = 0\\right]^2\\) but it is much more effective to instead fit it to another GB1 density, because GB1 allows us to travel back in time and express the posterior at the original prior’s time \\(t\\) instead of \\(t_2\\). I am delighted to acknowledge Robert Kern as the source of this insight.

The downside is that there appears to be no *analytical* way to fit a GB1 distribution to \\(p_{t_2} | x_{t_2} = 0\\), but a simple two-parameter numerical optimization suffices in all the cases I’ve looked at: find \\({β', δ'}\\) such that the least-squares error
\\[ \left( m_1 - \frac{B(α + δ', β')}{B(α, β')} \right)^2 + 
    \left( m_2 - \frac{B(α + 2 δ', β')}{B(α, β')} \right)^2, 
\\]
is ***minimized***. Here \\(m_1 = E\\left[p_{t_2} | x_{t_2} = 0\\right]\\) and \\(m_2 = E\\left[(p_{t_2} | x_{t_2} = 0\\right)^2]\\) are the first and second moments of the posterior for the failed quiz (see above), and \\(B(x,y)\\) is the [Beta](https://en.wikipedia.org/w/index.php?title=Beta_function&oldid=896304233#Relationship_between_gamma_function_and_beta_function) function. This numerical optimization is simply fitting the first two moments of the true posterior to the first two moments of candidate GB1 distributions, and trying to find the best GB1.

Having found such \\({β', δ'}\\), the posterior is approximately
\\[(p_{t_2} | x_{t_2} = 0) ≈ GB1(p_{t_2}; a=1/δ', b=1, p=α, q=β'),\\]
and we can time-travel this distribution backwards just like we did with the \\(x=1\\) case above, but not back to \\(t\\), but \\(t' = t_2 / δ'\\):
\\[(p_{t'} | x_{t_2} = 0) ≈ Beta(α, β').\\]

To summarize the update step: you started with a flashcard whose prior on recall at time \\(t\\) was \\(Beta(α, β)\\).
- For a successful quiz at time \\(t_2\\), set the new prior (the updated posterior) to \\(Beta(α + t_2/t, β)\\), at time \\(t\\) in the future.
- For the unsuccessful quiz, the new prior becomes \\(Beta(α, β')\\) at time \\(t_2 / δ'\\) in the future, where \\(β'\\) and \\(δ'\\) come from a two-dimensional numerical optimization above.

> **Note 1** To verify the failed quiz’s posterior moments in Mathematica: `Integrate[p^((a - t)/t)*(1 - p^(1/t))^(b - 1)*(1-p)*p^n, {p,0,1},  Assumptions -> a > 1 && b>1 && t>0 && n>=1]` gives the moment up to normalization by the marginal, that is, the numerator.
>
> `Integrate[p^((a - t)/t)*(1 - p^(1/t))^(b - 1)*(1-p), p]` and simplifying gives you the denominator (the marginal) of the moments. This also verifies that the previous integral works for the `n=0` case.
>
> **Note 2** Something we should note now is that the gamma function is a generalization of factorial—it’s a rapidly-growing function. With double-precision floats, \\(Γ(19) ≈ 6·10^{15}\\) has lost precision to the ones place, that is, `np.spacing(gamma(19)) == 1.0`). In this regime, which we regularly encounter particularly when over- and under-reviewing, addition and subtraction are risky. Ebisu takes care to factor these expressions to allow the use of log-gamma, [`expm1`](https://docs.scipy.org/doc/numpy/reference/generated/numpy.expm1.html), and [`logsumexp`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.misc.logsumexp.html), in order to minimize loss of precision.

## Source code

Before presenting the source code, I must somewhat apologetically explain a bit more about my workflow in writing and editing this document. I use the [Atom](https://atom.io) text editor with the [Hydrogen](https://atom.io/packages/hydrogen) plugin, which allows Atom to communicate with [Jupyter](http://jupyter.org/) kernels. Jupyter used to be called IPython, and is a standard protocol for programming REPLs to communicate with more modern applications like browsers or text editors. With this setup, I can write code in Atom and send it to a behind-the-scenes Python or Node.js or Haskell or Matlab REPL for evaluation, which sends back the result.

Hydrogen developer Lukas Geiger [recently](https://github.com/nteract/hydrogen/pull/637) added support for evaluating fenced code blocks in Markdown—a long-time dream of mine. This document is a Github-Flavored Markdown file to which I add fenced code blocks. Some of these code blocks I intend to just be demo code, and not end up in the Ebisu library proper, while the code below does need to go into `.py` files.

In order to untangle the code from the Markdown file to runnable files, I wrote a completely ad hoc undocumented Node.js script called [md2code.js](https://github.com/fasiha/ebisu/blob/gh-pages/md2code.js) which
- slurps the Markdown,
- looks for fenced code blocks that open with a comment indicating a file destination, e.g., `# export target.py`,
- prettifies Python with [Yapf](https://github.com/google/yapf), JavaScript with [clang-format](https://clang.llvm.org/docs/ClangFormatStyleOptions.html), etc.,
- dumps the code block contents into these files (appending after the first code block), and finally,
- updates the Markdown file itself with this prettified code.

All this enables me to stay in Atom, writing prose and editing/testing code by evaluating fenced code blocks, while also spitting out a proper Python or JavaScript library.

The major downside to this is that I cannot edit the untangled code files directly, and line numbers there don’t map to this document. I am tempted to append a commented-out line number in each untangled line…

### Core library

Python Ebisu contains a sub-module called `ebisu.alternate` which contains a number of alternative implementations of `predictRecall` and `updateRecall`. The `__init__` file sets up this module hierarchy.

```py
# export ebisu/__init__.py #
from .ebisu import *
from . import alternate
```

The above is in its own fenced code block because I don’t want Hydrogen to evaluate it. In Atom, I don’t work with the Ebisu module—I just interact with the raw functions.

Let’s present our Python implementation of the core Ebisu functions, `predictRecall` and `updateRecall`, and a couple of other related functions that live in the main `ebisu` module. All these functions consume a model encoding a Beta prior on recall probabilities at time \\(t\\), consisting of a 3-tuple containing \\((α, β, t)\\). I could have gone all object-oriented here but I chose to leave all these functions as stand-alone functions that consume and transform this 3-tuple because (1) I’m not an OOP devotee, and (2) I wanted to maximize the transparency of of this implementation so it can readily be ported to non-OOP, non-Pythonic languages.

> **Important** Note how none of these functions deal with *timestamps*. All time is captured in “time since last review”, and your external application has to assign units and store timestamps (as illustrated in the [Ebisu Jupyter Notebook](https://github.com/fasiha/ebisu/blob/gh-pages/EbisuHowto.ipynb)). This is a deliberate choice! Ebisu wants to know as *little* about your facts as possible.

In the [math section](#mean-and-variance-of-the-recall-probability-right-now) above we derived the mean recall probability at time \\(t_{now} = t · δ\\) given a model \\(α, β, t\\): \\(E[p_t^δ] = Γ(α + β) · Γ(α + δ) / (Γ(α) · Γ(α + β + δ))\\). There are no sums-of-gammas here, so this is readily computed using log-gamma routine (`gammaln` in Scipy) to avoid overflowing and precision-loss in `predictRecall` (🍏 below).

Two computational speedups are allowed for:
- we can skip the final `exp` that converts from the log-domain to the linear domain as long as we don’t need an actual probability (i.e., a number between 0 and 1). The output of the function will then be a “pseudo-probability” and can be compared to other “pseudo-probabilities” are returned by the function. Taking advantage of this can, for a representative case I have here, reduce the runtime from 5.69 µs (± 158 ns) to 4.01 µs (± 215 ns), a 1.4× speedup.
- \\(Γ(α + β) - Γ(α)\\) is independent of \\(t_{now}\\) and can be precomputed, for example, when the prior is first established. If this value is passed in, the runtime further reduces to 2.47 µs (± 162 ns), giving a total speedup of 2.3×, with no loss of accuracy. A helper function `cacheIndependent` is provided: 🥥 below.

```py
# export ebisu/ebisu.py #
def predictRecall(prior, tnow, exact=False, independent=None):
  """Expected recall probability now, given a prior distribution on it. 🍏

  `prior` is a tuple representing the prior distribution on recall probability
  after a specific unit of time has elapsed since this fact's last review.
  Specifically,  it's a 3-tuple, `(alpha, beta, t)` where `alpha` and `beta`
  parameterize a Beta distribution that is the prior on recall probability at
  time `t`.

  `tnow` is the *actual* time elapsed since this fact's most recent review.

  Optional keyword paramter `exact` makes the return value a probability, specifically, the expected recall probability `tnow` after the last review: a number between 0 and 1. If `exact` is false (the default), some calculations are skipped and the return value won't be a probability, but can still be compared against other values returned by this function. That is, if `predictRecall(prior1, tnow1, exact=True) < predictRecall(prior2, tnow2, exact=True)`, then it is guaranteed that `predictRecall(prior1, tnow1, exact=False) < predictRecall(prior2, tnow2, exact=False)`. The default is set to false for computational reasons.

  Optional keyword parameter `independent` is a precalculated number that is only dependent on `prior` and independent of `tnow`, allowing some computational speedup if cached ahead of time. It can be obtained with `ebisu.cacheIndependent`.

  See README for derivation.
  """
  from scipy.special import gammaln
  from numpy import exp
  alpha, beta, t = prior
  if not independent:
    independent = gammaln(alpha + beta) - gammaln(alpha)
  dt = tnow / t
  ret = gammaln(alpha + dt) - gammaln(alpha + beta + dt) + independent
  return exp(ret) if exact else ret


def cacheIndependent(prior):
  """Precompute a value to speed up `predictRecall`. 🥥

  Send the output of this function to `predictRecall`'s `independent` keyword argument.
  """
  from scipy.special import gammaln
  alpha, beta, t = prior
  return gammaln(alpha + beta) - gammaln(alpha)
```

Next is the implementation of `updateRecall` (🍌), which accepts
- a `model` (as above, represents the Beta prior on recall probability at one specific time since the fact’s last review),
- a quiz `result`: a truthy value meaning “passed quiz” and a false-ish value meaning “failed quiz”, and
- `tnow`, the actual time since last quiz that this quiz was administered.

and returns a *new* model, representing an updated Beta prior on recall probability, this time after `tnow` time has elapsed since a fact was quizzed.

**In case of successful quiz** `updateRecall` analytically computes the true posterior, which is GB1-distributed, and then exactly transforms these into a Beta distribution to yield the new model.

**In case of unsuccessful quiz**, `updateRecall` numerically fits a GB1 to the true posterior by matching the two distributions’ moments (via non-linear least squares). This approximate GB1 distribution is then exactly transformed back to a Beta distribution to yield the new model. Three helper functions are used:
- `gb1Moments`, which computes the moments of an arbitrary GB1 random variable,
- `failureMoments`, which computes the moments of the posterior upon a failed quiz, and
- `gb1ToBeta`, which translates a GB1 distribution on recall probability back in time to a pure Beta distribution.

```py
# export ebisu/ebisu.py #
def updateRecall(prior, result, tnow):
  """Update a prior on recall probability with a quiz result and time. 🍌

  `prior` is same as for `ebisu.predictRecall` and `predictRecallVar`: an object
  representing a prior distribution on recall probability at some specific time
  after a fact's most recent review.

  `result` is truthy for a successful quiz, false-ish otherwise.

  `tnow` is the time elapsed between this fact's last review and the review
  being used to update.

  Returns a new object (like `prior`) describing the posterior distribution of
  recall probability at `tnow`.
  """
  (alpha, beta, t) = prior
  dt = tnow / t
  if result:
    gb1 = (1.0 / dt, 1.0, alpha + dt, beta, tnow)
  else:
    import numpy as np

    mom = np.array(failureMoments(prior, result, tnow, num=2))

    def f(bd):
      b, d = bd
      this = np.array(gb1Moments(1 / d, 1., alpha, b, num=2))
      return this - mom

    from scipy.optimize import least_squares
    res = least_squares(f, [beta, dt], bounds=((1.01, 0), (np.inf, np.inf)))
    newBeta, newDelta = res.x
    gb1 = (1 / newDelta, 1, alpha, newBeta, tnow)

  return gb1ToBeta(gb1)


def gb1ToBeta(gb1):
  """Convert a GB1 model (five parameters: four GB1 parameters, time) to a Beta model
  
  `gb1: Tuple[float, float, float, float, float]`
  """
  return (gb1[2], gb1[3], gb1[4] * gb1[0])


def failureMoments(model, result, tnow, num=4, returnLog=True):
  """Moments of the posterior on recall at time `tnow` upon quiz failure
  
  - `model: Tuple[float, float, float]`
  - `result: bool`
  - `tnow: float`
  - `num: int`
  - `returnLog: bool`
  """
  a, b, t0 = model
  t = tnow / t0
  from scipy.special import gammaln, logsumexp
  from numpy import exp
  s = [gammaln(a + n * t) - gammaln(a + b + n * t) for n in range(num + 2)]
  marginal = logsumexp([s[0], s[1]], b=[1, -1])
  ret = [(logsumexp([s[n], s[n + 1]], b=[1, -1]) - marginal) for n in range(1, num + 1)]
  return ret if returnLog else [exp(x) for x in ret]


def gb1Moments(a, b, p, q, num=2, returnLog=True):
  """Raw moments of GB1, via Wikipedia
  
  `a: float, b: float, p: float, q: float, num: int, returnLog: bool`
  """
  from scipy.special import betaln
  import numpy as np
  bpq = betaln(p, q)
  logb = np.log(b)
  ret = [(h * logb + betaln(p + h / a, q) - bpq) for h in np.arange(1.0, num + 1)]
  return ret if returnLog else [np.exp(x) for x in ret]
```

Finally we have a couple more helper functions in the main `ebisu` namespace.

It is often useful to find out how much time has to elapse for memory to decay to a given recall probability. The time for memory to decay to 50% recall probability is the half-life. A quiz app might store the time when each quiz’s recall probability reaches 50%, 5%, 0.05%, …, as a computationally-efficient approximation to the exact recall probability. I am grateful to Robert Kern for contributing the `modelToPercentileDecay` function (🏀 below).

The least important function from a usage point of view is also the most important function for someone getting started with Ebisu: I call it `defaultModel` (🍗 below) and it simply creates a “model” object (a 3-tuple) out of the arguments it’s given. It’s included in the `ebisu` namespace to help developers who totally lack confidence in picking parameters: the only information it absolutely needs is an expected half-life, e.g., four hours or twenty-four hours or however long you expect a newly-learned fact takes to decay to 50% recall.

```py
# export ebisu/ebisu.py #
def modelToPercentileDecay(model, percentile=0.5):
  """When will memory decay to a given percentile? 🏀
  
  Use a root-finding routine in log-delta space to find the delta that
  will cause the GB1 distribution to have a mean of the requested quantile.
  Because we are using well-behaved normalized deltas instead of times, and
  owing to the monotonicity of the expectation with respect to delta, we can
  quickly scan for a rough estimate of the scale of delta, then do a finishing
  optimization to get the right value.
  """
  from scipy.special import betaln
  from scipy.optimize import root_scalar
  import numpy as np
  alpha, beta, t0 = model
  logBab = betaln(alpha, beta)
  logPercentile = np.log(percentile)

  def f(lndelta):
    logMean = betaln(alpha + np.exp(lndelta), beta) - logBab
    return logMean - logPercentile

  # Scan for a bracket.
  bracket_width = 6.0
  blow = -bracket_width / 2.0
  bhigh = bracket_width / 2.0
  flow = f(blow)
  fhigh = f(bhigh)
  while flow > 0 and fhigh > 0:
    # Move the bracket up.
    blow = bhigh
    flow = fhigh
    bhigh += bracket_width
    fhigh = f(bhigh)
  while flow < 0 and fhigh < 0:
    # Move the bracket down.
    bhigh = blow
    fhigh = flow
    blow -= bracket_width
    flow = f(blow)

  assert flow > 0 and fhigh < 0

  sol = root_scalar(f, bracket=[blow, bhigh])
  t1 = np.exp(sol.root) * t0
  return t1


def defaultModel(t, alpha=3.0, beta=None):
  """Convert recall probability prior's raw parameters into a model object. 🍗

  `t` is your guess as to the half-life of any given fact, in units that you
  must be consistent with throughout your use of Ebisu.

  `alpha` and `beta` are the parameters of the Beta distribution that describe
  your beliefs about the recall probability of a fact `t` time units after that
  fact has been studied/reviewed/quizzed. If they are the same, `t` is a true
  half-life, and this is a recommended way to create a default model for all
  newly-learned facts. If `beta` is omitted, it is taken to be the same as
  `alpha`.
  """
  return (alpha, beta or alpha, t)
```

I would expect all the functions above to be present in all implementations of Ebisu:
- `predictRecall`, aided by a public helper function `cacheIndependent`,
- `updateRecall`, aided by private helper functions `gb1ToBeta`, `failureMoments`, and `gb1Moments`,
- `modelToPercentileDecay`, and
- `defaultModel`.

The functions in the following section are either for illustrative or debugging purposes.

### Miscellaneous functions
I wrote a number of other functions that help provide insight or help debug the above functions in the main `ebisu` workspace but are not necessary for an actual implementation. These are in the `ebisu.alternate` submodule and not nearly as much time has been spent on polish or optimization as the above core functions.

First, a helper function that fits a Beta random variable to a mean and variance.
```py
# export ebisu/alternate.py #
def _meanVarToBeta(mean, var):
  """Fit a Beta distribution to a mean and variance. 🏈"""
  # [betaFit] https://en.wikipedia.org/w/index.php?title=Beta_distribution&oldid=774237683#Two_unknown_parameters
  tmp = mean * (1 - mean) / var - 1
  alpha = mean * tmp
  beta = (1 - mean) * tmp
  return alpha, beta
```

`predictRecallMode` and `predictRecallMedian` return the mode and median of the recall probability prior rewound or fast-forwarded to the current time. That is, they return the mode/median of the random variance \\(p_t^δ\\) whose mean is returned by `predictRecall` (🍏 above). Recall that \\(δ = t / t_{now}\\).

The mode has an analytical expression, and while it is more meaningful than the mean, the distribution can blow up to infinity at 0 or 1 when \\(δ\\) is either much smaller or much larger than 1, in which case the analytical expression may yield nonsense, so a number of not-very-rigorous checks are in place to attempt to detect this.

I could not find a closed-form expression for the median of \\(p_t^δ\\), so I use a bracketed root search (Brent’s algorithm) on the cumulative distribution function (the CDF), for which Wolfram Alpha can yield an analytical expression. This can get numerically burdensome, which is unacceptable because one may need to predict the recall probability for thousands of facts. For these reasons, although I would have preferred to make `predictRecall` evaluate the mode or median, I made it return the mean.

`predictRecallMonteCarlo` is the simplest function. It evaluates the mean, variance, mode (via histogram), and median of \\(p_t^δ\\) by drawing samples from the Beta prior on \\(p_t\\) and raising them to the \\(δ\\)-power. The unit tests for `predictRecall` and `predictRecallVar` in the next section use this Monte Carlo to test both derivations and implementations. While fool-proof, Monte Carlo simulation is obviously far too computationally-burdensome for regular use.

```py
# export ebisu/alternate.py #
import numpy as np


def predictRecallMode(prior, tnow):
  """Mode of the immediate recall probability.

  Same arguments as `ebisu.predictRecall`, see that docstring for details. A
  returned value of 0 or 1 may indicate divergence.
  """
  # [1] Mathematica: `Solve[ D[p**((a-t)/t) * (1-p**(1/t))**(b-1), p] == 0, p]`
  alpha, beta, t = prior
  dt = tnow / t
  pr = lambda p: p**((alpha - dt) / dt) * (1 - p**(1 / dt))**(beta - 1)

  # See [1]. The actual mode is `modeBase ** dt`, but since `modeBase` might
  # be negative or otherwise invalid, check it.
  modeBase = (alpha - dt) / (alpha + beta - dt - 1)
  if modeBase >= 0 and modeBase <= 1:
    # Still need to confirm this is not a minimum (anti-mode). Do this with a
    # coarse check of other points likely to be the mode.
    mode = modeBase**dt
    modePr = pr(mode)

    eps = 1e-3
    others = [
        eps, mode - eps if mode > eps else mode / 2, mode + eps if mode < 1 - eps else
        (1 + mode) / 2, 1 - eps
    ]
    otherPr = map(pr, others)
    if max(otherPr) <= modePr:
      return mode
  # If anti-mode detected, that means one of the edges is the mode, likely
  # caused by a very large or very small `dt`. Just use `dt` to guess which
  # extreme it was pushed to. If `dt` == 1.0, and we get to this point, likely
  # we have malformed alpha/beta (i.e., <1)
  return 0.5 if dt == 1. else (0. if dt > 1 else 1.)


def predictRecallMedian(prior, tnow, percentile=0.5):
  """Median (or percentile) of the immediate recall probability.

  Same arguments as `ebisu.predictRecall`, see that docstring for details.

  An extra keyword argument, `percentile`, is a float between 0 and 1, and
  specifies the percentile rather than 50% (median).
  """
  # [1] `Integrate[p**((a-t)/t) * (1-p**(1/t))**(b-1) / t / Beta[a,b], p]`
  # and see "Alternate form assuming a, b, p, and t are positive".
  from scipy.special import betaincinv
  alpha, beta, t = prior
  dt = tnow / t
  return betaincinv(alpha, beta, percentile)**dt


def predictRecallMonteCarlo(prior, tnow, N=1000 * 1000):
  """Monte Carlo simulation of the immediate recall probability.

  Same arguments as `ebisu.predictRecall`, see that docstring for details. An
  extra keyword argument, `N`, specifies the number of samples to draw.

  This function returns a dict containing the mean, variance, median, and mode
  of the current recall probability.
  """
  import scipy.stats as stats
  alpha, beta, t = prior
  tPrior = stats.beta.rvs(alpha, beta, size=N)
  tnowPrior = tPrior**(tnow / t)
  freqs, bins = np.histogram(tnowPrior, 'auto')
  bincenters = bins[:-1] + np.diff(bins) / 2
  return dict(
      mean=np.mean(tnowPrior),
      median=np.median(tnowPrior),
      mode=bincenters[freqs.argmax()],
      var=np.var(tnowPrior))
```

Next we have a Monte Carlo approache to `updateRecall` (🍌 above), the deceptively-simple `updateRecallMonteCarlo`. Like `predictRecallMonteCarlo` above, it draws samples from the Beta distribution in `model` and propagates them through Ebbinghaus’ forgetting curve to the time specified. To model the likelihood update from the quiz result, it assigns weights to each sample—each weight is that sample’s probability according to the Bernoulli likelihood. (This is equivalent to multiplying the prior with the likelihood—and we needn’t bother with the marginal because it’s just a normalizing factor which would scale all weights equally. I am grateful to [mxwsn](https://stats.stackexchange.com/q/273221/31187) for suggesting this elegant approach.) Since our real GB1-based `updateRecall` above returns posteriors at potentially different times, `updateRecallMonteCarlo` optionally takes a `tback` time to rewind or forward-wind the Monte Carlo ensemble, again through Ebbinghaus’ exponential decay curve. Finally, the ensemble is collapsed to a weighted mean and variance and converted to a Beta distribution.

```py
# export ebisu/alternate.py #
def updateRecallMonteCarlo(prior, result, tnow, tback, N=10 * 1000 * 1000):
  """Update recall probability with quiz result via Monte Carlo simulation.

  Same arguments as `ebisu.updateRecall`, see that docstring for details.

  An extra keyword argument `N` specifies the number of samples to draw.
  """
  # [bernoulliLikelihood] https://en.wikipedia.org/w/index.php?title=Bernoulli_distribution&oldid=769806318#Properties_of_the_Bernoulli_Distribution, third equation
  # [weightedMean] https://en.wikipedia.org/w/index.php?title=Weighted_arithmetic_mean&oldid=770608018#Mathematical_definition
  # [weightedVar] https://en.wikipedia.org/w/index.php?title=Weighted_arithmetic_mean&oldid=770608018#Weighted_sample_variance
  import scipy.stats as stats

  alpha, beta, t = prior

  tPrior = stats.beta.rvs(alpha, beta, size=N)
  tnowPrior = tPrior**(tnow / t)

  # This is the Bernoulli likelihood [bernoulliLikelihood]
  weights = (tnowPrior)**result * ((1 - tnowPrior)**(1 - result))

  # Now propagate this posterior to the tback
  tbackPrior = tnowPrior**(tback / tnow)

  # See [weightedMean]
  weightedMean = np.sum(weights * tbackPrior) / np.sum(weights)
  # See [weightedVar]
  weightedVar = np.sum(weights * (tbackPrior - weightedMean)**2) / np.sum(weights)

  newAlpha, newBeta = _meanVarToBeta(weightedMean, weightedVar)

  return newAlpha, newBeta, tback
```

That’s it—that’s all the code in the `ebisu` module!

### Test code
I use the built-in `unittest`, and I can run all the tests from Atom via Hydrogen/Jupyter but for historic reasons I don’t want Jupyter to deal with the `ebisu` namespace, just functions (since most of these functions and tests existed before the module’s layout was decided). So the following is in its own fenced code block that I don’t evaluate in Atom.

```py
# export ebisu/tests/test_ebisu.py
from ebisu import *
from ebisu.alternate import *
```

In these unit tests, I compare
- `predictRecall` and `predictRecallVar` against `predictRecallMonteCarlo`, and
- `updateRecall` against `updateRecallMonteCarlo`.

I also want to make sure that `predictRecall` and `updateRecall` both produce sane values when extremely under- and over-reviewing, i.e., immediately after review as well as far into the future. And we should also exercise `modelToPercentileDecay`.

For testing `updateRecall`, since all functions return a Beta distribution, I compare the resulting distributions in terms of [Kullback–Leibler divergence](https://en.wikipedia.org/w/index.php?title=Beta_distribution&oldid=774237683#Quantities_of_information_.28entropy.29) (actually, the symmetric distance version), which is a nice way to measure the difference between two probability distributions. There is also a little unit test for my implementation for the KL divergence on Beta distributions.

For testing `predictRecall`, I compare means using relative error, \\(|x-y| / |y|\\).

For both sets of functions, a range of \\(δ = t_{now} / t\\) and both outcomes of quiz results (true and false) are tested to ensure they all produce the same answers.

Often the unit tests fails because the tolerances are a little tight, and the random number generator seed is variable, which leads to errors exceeding thresholds. I actually prefer to see these occasional test failures because it gives me confidence that the thresholds are where I want them to be (if I set the thresholds too loose, and I somehow accidentally greatly improved accuracy, I might never know). However, I realize it can be annoying for automated tests or continuous integration systems, so I am open to fixing a seed and fixing the error threshold for it.

One note: the unit tests udpate a global database of `testpoints` being tested, which can be dumped to a JSON file for comparison against other implementations.

```py
# export ebisu/tests/test_ebisu.py
import unittest
import numpy as np


def relerr(dirt, gold):
  return abs(dirt - gold) / abs(gold)


def maxrelerr(dirts, golds):
  return max(map(relerr, dirts, golds))


def klDivBeta(a, b, a2, b2):
  """Kullback-Leibler divergence between two Beta distributions in nats"""
  # Via http://bariskurt.com/kullback-leibler-divergence-between-two-dirichlet-and-beta-distributions/
  from scipy.special import gammaln, psi
  import numpy as np
  left = np.array([a, b])
  right = np.array([a2, b2])
  return gammaln(sum(left)) - gammaln(sum(right)) - sum(gammaln(left)) + sum(
      gammaln(right)) + np.dot(left - right,
                               psi(left) - psi(sum(left)))


def kl(v, w):
  return (klDivBeta(v[0], v[1], w[0], w[1]) + klDivBeta(w[0], w[1], v[0], v[1])) / 2.


testpoints = []


class TestEbisu(unittest.TestCase):

  def test_predictRecallMedian(self):
    model0 = (4.0, 4.0, 1.0)
    model1 = updateRecall(model0, False, 1.0)
    model2 = updateRecall(model1, True, 0.01)
    ts = np.linspace(0.01, 4.0, 81.0)
    qs = (0.05, 0.25, 0.5, 0.75, 0.95)
    for t in ts:
      for q in qs:
        self.assertGreater(predictRecallMedian(model2, t, q), 0)

  def test_kl(self):
    # See https://en.wikipedia.org/w/index.php?title=Beta_distribution&oldid=774237683#Quantities_of_information_.28entropy.29 for these numbers
    self.assertAlmostEqual(klDivBeta(1., 1., 3., 3.), 0.598803, places=5)
    self.assertAlmostEqual(klDivBeta(3., 3., 1., 1.), 0.267864, places=5)

  def test_prior(self):
    "test predictRecall vs predictRecallMonteCarlo"

    def inner(a, b, t0):
      global testpoints
      for t in map(lambda dt: dt * t0, [0.1, .99, 1., 1.01, 5.5]):
        mc = predictRecallMonteCarlo((a, b, t0), t, N=100 * 1000)
        mean = predictRecall((a, b, t0), t, exact=True)
        self.assertLess(relerr(mean, mc['mean']), 5e-2)
        testpoints += [['predict', [a, b, t0], [t], dict(mean=mean)]]

    inner(3.3, 4.4, 1.)
    inner(34.4, 34.4, 1.)

  def test_posterior(self):
    "Test updateRecall via updateRecallMonteCarlo"

    def inner(a, b, t0, dts):
      global testpoints
      for t in map(lambda dt: dt * t0, dts):
        for x in [False, True]:
          msg = 'a={},b={},t0={},x={},t={}'.format(a, b, t0, x, t)
          an = updateRecall((a, b, t0), x, t)
          mc = updateRecallMonteCarlo((a, b, t0), x, t, an[2], N=100 * 1000)
          self.assertLess(kl(an, mc), 5e-3, msg=msg + ' an={}, mc={}'.format(an, mc))

          testpoints += [['update', [a, b, t0], [x, t], dict(post=an)]]

    inner(3.3, 4.4, 1., [0.1, 1., 9.5])
    inner(34.4, 3.4, 1., [0.1, 1., 5.5, 50.])

  def test_update_then_predict(self):
    "Ensure #1 is fixed: prediction after update is monotonic"
    future = np.linspace(.01, 1000, 101)

    def inner(a, b, t0, dts):
      for t in map(lambda dt: dt * t0, dts):
        for x in [False, True]:
          msg = 'a={},b={},t0={},x={},t={}'.format(a, b, t0, x, t)
          newModel = updateRecall((a, b, t0), x, t)
          predicted = np.vectorize(lambda tnow: predictRecall(newModel, tnow))(future)
          self.assertTrue(
              np.all(np.diff(predicted) < 0), msg=msg + ' predicted={}'.format(predicted))

    inner(3.3, 4.4, 1., [0.1, 1., 9.5])
    inner(34.4, 3.4, 1., [0.1, 1., 5.5, 50.])

  def test_halflife(self):
    "Exercise modelToPercentileDecay"
    percentiles = np.linspace(.01, .99, 101)

    def inner(a, b, t0, dts):
      for t in map(lambda dt: dt * t0, dts):
        msg = 'a={},b={},t0={},t={}'.format(a, b, t0, t)
        ts = np.vectorize(lambda p: modelToPercentileDecay((a, b, t), p))(percentiles)
        self.assertTrue(np.all(np.diff(ts) < 0), msg=msg + ' ts={}'.format(ts))

    inner(3.3, 4.4, 1., [0.1, 1., 9.5])
    inner(34.4, 3.4, 1., [0.1, 1., 5.5, 50.])


if __name__ == '__main__':
  unittest.TextTestRunner().run(unittest.TestLoader().loadTestsFromModule(TestEbisu()))

  with open("test.json", "w") as out:
    import json
    out.write(json.dumps(testpoints))
```

That `if __name__ == '__main__'` is for running the test suite in Atom via Hydrogen/Jupyter. I actually use nose to run the tests, e.g., `python3 -m nose` (which is wrapped in an npm script: if you look in `package.json` you’ll see that `npm test` will run the eqivalent of `node md2code.js && python3 -m "nose"`: this Markdown file is untangled into Python source files first, and then nose is invoked).

## Demo codes

The code snippets here are intended to demonstrate some Ebisu functionality.

### Visualizing half-lives

The first snippet produces the half-life plots shown above, and included below, scroll down.

```py
import scipy.stats as stats
import numpy as np
import matplotlib.pyplot as plt

plt.style.use('ggplot')
plt.rcParams['svg.fonttype'] = 'none'

t0 = 7.

ts = np.arange(1, 301.)
ps = np.linspace(0, 1., 200)
ablist = [3, 12]

plt.close('all')
plt.figure()
[
    plt.plot(
        ps, stats.beta.pdf(ps, ab, ab) / stats.beta.pdf(.5, ab, ab), label='α=β={}'.format(ab))
    for ab in ablist
]
plt.legend(loc=2)
plt.xticks(np.linspace(0, 1, 5))
plt.title('Confidence in recall probability after one half-life')
plt.xlabel('Recall probability after one week')
plt.ylabel('Prob. of recall prob. (scaled)')
plt.savefig('figures/models.svg')
plt.savefig('figures/models.png', dpi=300)
plt.show()

plt.figure()
ax = plt.subplot(111)
plt.axhline(y=t0, linewidth=1, color='0.5')
[
    plt.plot(
        ts,
        list(map(lambda t: modelToPercentileDecay(updateRecall((a, a, t0), xobs, t)), ts)),
        marker='x' if xobs == 1 else 'o',
        color='C{}'.format(aidx),
        label='α=β={}, {}'.format(a, 'pass' if xobs == 1 else 'fail'))
    for (aidx, a) in enumerate(ablist)
    for xobs in [1, 0]
]
plt.legend(loc=0)
plt.title('New half-life (previously {:0.0f} days)'.format(t0))
plt.xlabel('Time of test (days after previous test)')
plt.ylabel('Half-life (days)')
plt.savefig('figures/halflife.svg')
plt.savefig('figures/halflife.png', dpi=300)
plt.show()
```

![figures/models.png](figures/models.png)

![figures/halflife.png](figures/halflife.png)

### Why we work with random variables

This second snippet addresses a potential approximation which isn’t too accurate but might be useful in some situations. The function `predictRecall` (🍏 above) in exaxct mode evaluates the log-gamma function four times and an `exp` once. One may ask, why not use the half-life returned by `modelToPercentileDecay` and Ebbinghaus’ forgetting curve, thereby approximating the current recall probability for a fact as `2 ** (-tnow / modelToPercentileDecay(model))`? While this is likely more computationally efficient (after computing the half-life up-front), it is also less precise:

```py
ts = np.linspace(1, 41)

modelA = updateRecall((3., 3., 7.), 1, 15.)
modelB = updateRecall((12., 12., 7.), 1, 15.)
hlA = modelToPercentileDecay(modelA)
hlB = modelToPercentileDecay(modelB)

plt.figure()
[
    plt.plot(ts, predictRecall(model, ts, exact=True), '.-', label='Model ' + label, color=color)
    for model, color, label in [(modelA, 'C0', 'A'), (modelB, 'C1', 'B')]
]
[
    plt.plot(ts, 2**(-ts / halflife), '--', label='approx ' + label, color=color)
    for halflife, color, label in [(hlA, 'C0', 'A'), (hlB, 'C1', 'B')]
]
# plt.yscale('log')
plt.legend(loc=0)
plt.ylim([0, 1])
plt.xlabel('Time (days)')
plt.ylabel('Recall probability')
plt.title('Predicted forgetting curves (halflife A={:0.0f}, B={:0.0f})'.format(hlA, hlB))
plt.savefig('figures/forgetting-curve.svg')
plt.savefig('figures/forgetting-curve.png', dpi=300)
plt.show()
```

![figures/forgetting-curve.png](figures/forgetting-curve.png)

This plot shows `predictRecall`’s fully analytical solution for two separate models over time as well as this approximation: model A has half-life of eleven days while model B has half-life of 7.9 days. We see that the approximation diverges a bit from the true solution.

This also indicates that placing a prior on recall probabilities and propagating that prior through time via Ebbinghaus results in a *different* curve than Ebbinghaus’ exponential decay curve. This surprising result can be seen as a consequence of [Jensen’s inequality](https://en.wikipedia.org/wiki/Jensen%27s_inequality), which says that \\(E[f(p)] ≥ f(E[p])\\) when \\(f\\) is convex, and that the opposite is true if it is concave. In our case, \\(f(p) = p^δ\\), for `δ = t / halflife`, and Jensen requires that the accurate mean recall probability is greater than the approximation for times greater than the half-life, and less than otherwise. We see precisely this for both models, as illustrated in this plot of just their differences:

```py
plt.figure()
ts = np.linspace(1, 14)

plt.axhline(y=0, linewidth=3, color='0.33')
plt.plot(ts, predictRecall(modelA, ts, exact=True) - 2**(-ts / hlA), label='Model A')
plt.plot(ts, predictRecall(modelB, ts, exact=True) - 2**(-ts / hlB), label='Model B')
plt.gcf().subplots_adjust(left=0.15)
plt.legend(loc=0)
plt.xlabel('Time (days)')
plt.ylabel('Difference')
plt.title('Expected recall probability minus approximation')
plt.savefig('figures/forgetting-curve-diff.svg')
plt.savefig('figures/forgetting-curve-diff.png', dpi=300)
plt.show()
```

![figures/forgetting-curve-diff.png](figures/forgetting-curve-diff.png)

I think this speaks to the surprising nature of random variables and the benefits of handling them rigorously, as Ebisu seeks to do.

### Moving Beta distributions through time
Below is the code to show the histograms on recall probability two days, a week, and three weeks after the last review:
```py
def generatePis(deltaT, alpha=12.0, beta=12.0):
  import scipy.stats as stats

  piT = stats.beta.rvs(alpha, beta, size=50 * 1000)
  piT2 = piT**deltaT
  plt.hist(piT2, bins=20, label='δ={}'.format(deltaT), alpha=0.25, normed=True)


[generatePis(p) for p in [0.3, 1., 3.]]
plt.xlabel('p (recall probability)')
plt.ylabel('Probability(p)')
plt.title('Histograms of p_t^δ for different δ')
plt.legend(loc=0)
plt.savefig('figures/pidelta.svg')
plt.savefig('figures/pidelta.png', dpi=150)
plt.show()
```

![figures/pidelta.png](figures/pidelta.png)


## Requirements for building all aspects of this repo

- Python
    - scipy, numpy
    - nose for tests
- [Pandoc](http://pandoc.org)
- [pydoc-markdown](https://pypi.python.org/pypi/pydoc-markdown)

**Implementation ideas** Lua, Erlang, Elixir, Red, F#, OCaml, Reason, PureScript, JS, TypeScript, Rust, … Postgres (w/ or w/o GraphQL), SQLite, LevelDB, Redis, Lovefield, …

## Acknowledgements

Many thanks to [mxwsn and commenters](https://stats.stackexchange.com/q/273221/31187) as well as [jth](https://stats.stackexchange.com/q/272834/31187) for their advice and patience with my statistical incompetence.

Many thanks also to Drew Benedetti for reviewing this manuscript.

John Otander’s [Modest CSS](http://markdowncss.github.io/modest/) is used to style the Markdown output.
















