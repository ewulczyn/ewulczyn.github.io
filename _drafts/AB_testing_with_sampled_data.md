---
layout: post
title: AB Testing With Sampled Data
---

The Wikimedia Foundation (WMF) is in the unique position that we have a ton of
traffic and very little resources. When AB testing banners, we get so many
impressions we have to sample the number of impressions we record. This
complicates estimating the conversion rate for our banners. In traditional AB
testing, we model the conversion rate as a bernoulli random variable. When
running an AB test with unsampled impression counts, all the uncertainty in true
conversion rates comes from the fact that a conversion is a random event and we
have a finite number of impressions. However, if we sample the number of
impressions, we also need to deal with the uncertainty in the number of banners
we actually served! What follows is an extension the Bayesian hypothesis test
for bernoulli data that accounts for the additional uncertainty introduced by
sampling.


## Why do we sample impressions?

Before I dig into the theory, let me explain exactly what I mean when I say the
impression numbers are sampled and why they are sampled. During a fundraising
campaign, we try to serve a banner on every pageview. When the page loads, it
fetches the current banner for the experimental condition the client is in.
Depending on client side cookies set by previous fundraising banners, the banner
may or may not display. For example, if the client closed a banner in the last
two weeks, the banner may choose to hide itself. Recording whether there was an
impression requires sending an extra request back to our servers. The operations
team deems it unacceptable to send such an banner impression logging request for
every pageview. So instead, we send the request back with probability \\(r\\).

At this point I assume you are scratching your head. Why can't we just decide if
the banner will be displayed server side and avoid sending the request. Well,
the WMF infrastructure is designed to serve requests for non logged-in users out
of cache, even banners. As such it is not possible to evaluate if the banner
should be shown based on the client's fundraising cookies server-side. I'm sure
there are ways to get around sampling, but the fact that impressions are sampled
is a constraint I currently face.

## Estimating the number of Impressions

The first challenge in developing theory for AB testing under these
circumstances is to estimate how many impressions there really were \\(N\\) given
the number of logged impression records \\(n\\). To make this formal, consider a
random process in which we randomly record an event with fixed, known
probability \\(r\\). Say we have \\(n\\) recorded events. Lets work out \\(P(N | n)\\), the
distribution over the true number of events \\(N\\), given the sampling rate \\(r\\) and
the number of recorded events \\(n\\).

If you remember your discrete distributions, this might sound reminiscent
of a negative binomial random variable. That is almost correct. The negative
binomial random variable corresponds to the number of attempts \\(N\\) until the \\(n\\)th
success. In our scenario, we don't know how many impressions there where after
the last recorded impression (success). What follows is the derivation a
distribution that does not assume the last event was a success. For situations where you have so much data that you have
to sample, it turns out to be very similar to the negative binomial. So if you
don't enjoy proofs, feel free to skip to the next section.

### Deriving the Inverse Binomial

We know that \\(P(n \| N)\\) ~  Binomial(N, r). Applying Bayes rule and using a uniform prior over \\(N\\), we get:

$$
P(N=j | n) = 
$$

$$
\frac{P(n | N=j)P(N=j)}{\sum\limits_{i=n}^\infty{P(n|N=i)P(N=i)}}=
$$

$$
 \frac{P(n | N=j)}{\sum\limits_{i=n}^\infty{P(n|N=i)}} = 
$$

$$
\frac{\binom{j}{n} r^{n} (1-r)^{j-n}}{\sum\limits_{i=n}^\infty{\binom{i}{n}r^{n}(1-r)^{i-n}}}
$$



Lets work out the infinite sum in the denominator to get the normalizaton term:


$$
\sum\limits_{i=n}^\infty{\binom{i}{n} r^{n} (1-r)^{i-n}} =
$$

$$ 
\sum\limits_{i^\prime=n^\prime}^\infty{\binom{i^\prime -1}{n^\prime -1}r^{n^\prime -1}(1-r)^{i^\prime-n^\prime}} =
$$

$$
 \frac{1}{r} \sum\limits_{i^\prime=n^\prime}^\infty{\binom{i^\prime -1}{
n^\prime -1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}} =
$$

$$
\frac{1}{r}
$$


In the first step, we make a transformation of variables. We set \\(n = n^\prime -1\\) and
\\(i = i^\prime -1\\). In the second we make use of the fact that

$$
\binom{i^\prime -1 }{n^\prime -1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}
$$

is the pmf of the negative binomial distribution and sums to 1. So, we arrive at our final result:


$$
P(N | r, n) = \begin{cases}
0, &{\rm if}\ N\ \text{less than}\ n  \\
\binom{N}{n} r^{n+1} (1-r)^{N-n}, &{\rm otherwise}
\end{cases}
$$

Lets refer to this newly derived distribution as the "inverse binomial
distribution"


### The Expectation of the Inverse Binomial

Next, lets work out the expectation:


$$
E[N] =
$$

$$
\sum\limits_{i=n}^\infty{i*\binom{i}{n} r^{n+1} (1-r)^{i-n}} =
$$

$$
\sum\limits_{i^\prime=n^\prime}^\infty{(i^\prime -1)*\binom{i^\prime -1}{n^\prime -1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}} =
$$

<div>
$$
\sum\limits_{i^\prime=n^\prime}^\infty{i^\prime * \binom{i^\prime -1}{n^\prime-1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}} - \sum\limits_{i^\prime=n^\prime}^\infty{\binom{i^\prime -1}{n^\prime -1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}}=
$$
</div>

$$
\frac{n^\prime}{r} - 1 =
$$

$$
\frac{n + 1}{r} -1
$$

What a nice result! In step 3 we make use of the fact that

$$
\sum\limits_{i^\prime=n^\prime}^\infty{i^\prime * \binom{i^\prime -1}{n^\prime-1} r^{n^\prime} (1-r)^{i^\prime-n^\prime}}
$$

is the expectation of a negative binomial distribution. To get a sense for how the inverse-binomial differs from the negative-binomial, lets assume we observe one success and compare the expectations of the distributions as a function of the smapling rate r.



![_config.yml]({{ site.baseurl }}/ipython/AB_testing_with_sampled_data/AB_testing_with_sampled_data_files/AB_testing_with_sampled_data_16_0.png)


As mentioned above, the negative binomial random variable is the expected number
of coin tosses before the nth heads. This is pretty close to our inverse
binomial random variable. The difference is that we are not guaranteed that
the last of our data points was sampled. In fact there could have been arbitrarily many data
points that we did not sample after the last point we did sample. Hence it seems reasonable that
the expectation of the negative binomial distribution is a lower bound
for the inverse binomial distribution. Furthermore, the expectation of the
negative binomial distribution, where we act as if we saw one more sample than
we really did, is an upper bound for our inverse binomial rv. Depending on the sampling rate, the inverse binomial, swings between it supper and lower bounds. As \\(n\\) increases, the difference in expectation between the distributions becomes less and less significant. Hence, for AB tests with large data volumes, we may as well
assume that our impressions is negative binomial.

## The Traditional Bayesian AB Test

In traditional Bayesian AB testing, we model a conversion as bernoulli random
variable. We also model the conversion rate \\(p\\) as a random variable following a
beta distribution. Before observing any data, we model \\(p\\) as \\(beta(a\_{prior}, b\_{prior})\\).
 Usually, we set \\(a\_{prior}=b\_{prior}=1\\), which corresponds to a
uniform distribution over the interval \\([0,1]\\). After running a banner we count
how many impressions led to a donation (successes) and how many impressions did
not (failures).


Then we update the distribution over \\(p\\) to be  \\(beta(a\_{prior} + successes,
b\_{prior} + failures)\\). Once we have the distributions for the conversion rate
for each of the banners, we can use numerical methods to arrive at the
distribution over functions of the conversion rates. We will be most interested
in the distribution over the percent difference in conversion rates. For a more
thorough introduction to Bayesian AB testing, check out this
[post](http://engineering.richrelevance.com/bayesian-ab-tests/).

The code below demonstrates running a traditional Bayesian AB test, where
the number of impressions are known in advance. In particular, we compute a 100-\\(a\\)% confidence
interval for the percent lift that B gives over A. 

~~~python
from numpy.random import binomial, beta

# True donation rates
p_A = 0.5
p_B = 0.6 # B has a 20% lift over A


# True number of impressions per banner
N_A = 1000
N_B = 1200

#simulate Running the banner
successes_A = binomial(N_A, p_A)
successes_B = binomial(N_B, p_B)

# form posterior distribution over the lift  banner B gives over banner A
failures_A = N_A -successes_A
failures_B = N_B -successes_B

N_samples = 20000
p_A_posterior = beta(1+successes_A, 1+failures_A, size = N_samples)
p_B_posterior =  beta(1+successes_B, 1+failures_B, size = N_samples)

lift_distribution = (p_B_posterior-p_A_posterior)/p_A_posterior
lift_distribution = lift_distribution *100

# 1-a Credible interval for the lift that B ahs over A
a = 10
lower = np.percentile(lift_distribution, a/2)
upper = np.percentile(lift_distribution, 100- a/2)
print "%d%% Confidence Interval for the lift B has over A: (%f, %f)" %(100-a, lower, upper)

90% Confidence Interval for the lift B has over A: (13.920149, 28.936362)
~~~


## The Traditional Method Breaks Down with Sampled Data

The traditional bayesian method described above won't work for the case of sampled impressions.
When we sample the number of impressions, we still know how many successes we got, since we process the
payment for every donation and the donation hits our accounts. But we do not how
many impressions did not lead to a donation (failures). We will have to find a
way of incorporate the uncertainty we have in the number of failed impressions
into the traditional method.

Note: How do we get the number of failed impressions when all the have is the
number of observed impressions? We will need to generate an ID on each
impression and log that ID with each impression and each donation. Then we can
get the number of observed impressions that did not lead to a donation by
removing impressions with the same ID as a donation.

Lets start with a naive approach of simply using the expected number of failed
impression given the sampling rate r and the number of observed failed
impressions n: \\(\frac{(n+1)}{r} - 1\\). We derived this result above. We will be interested in measuring how
often our 100-\\(a\\)% confidence intervals for the lift that banner B has over banner
A cover the true lift. We can do this by repeatedly simulating AB tests and measuring
what fraction of the time our lift intervals cover the true lift. More specifically, we will choose the true donation rate for both banners, 
the sampling rate for failed impressions, the length of the experiment \\(N\\)
and the confidence level of our confidence intervals. On each banner impression, we first flip a coin with probability \\(p\\) to determine if the 
impression leads to a donation. If it does, we record the donation. If it does not, 
we flip a coin with probability \\(r\\) to determine whether to record failed impression.
We then compute the confidence interval for the lift banner B gives over banner A, based on
the number of success and the fixed estimate of the total number of failures. We record how often
the true lift, which we know because we choose the click through rates of the banners, is 
covered by our confidence interval. If the method is correct, the intervals we construct will cover the 
true lift 100-\\(a\\)% of the time.


~~~ python
from numpy.random import negative_binomial, binomial, beta

def inverse_binomial_expectation(n, r):
    return (n+1)/r -1

def get_p_dist_naive(successes, observed_failures, r):
    """
    get posterior distribution over the conversion
    rate p, when using the expected number of failed impressions
    given a sampling rate of r and n observed failures
    """
    estimated_failures = int(inverse_binomial_expectation(observed_failures, r))
    return beta(1+successes, 1+estimated_failures, size = 20000)

def get_lift_confidence_interval(p_A_posterior, p_B_posterior, conf):
    """
    compute a credible interval over the lift that
    banner A has over banner B given the data
    """
    a = (100-conf) 
    lift_distribution = (p_B_posterior-p_A_posterior)/p_A_posterior
    lift_distribution = lift_distribution *100
    lower = np.percentile(lift_distribution, a/2)
    upper = np.percentile(lift_distribution, 100- a/2)
    return (lower, upper)


def lift_interval_covers_true_lift(interval,true_lift):
    """
    Check if the credible interval covers the true parameter
    """
    if interval[1] < true_lift:
        return 0
    if interval[0] > true_lift:
        return 0
    return 1       
~~~


~~~ python
def measure_coverage(N, p_A, lift, r, get_p_dist, iters = 500, conf = 90):
    """
    This function simulates iters AB tests and measures the coverage of the 
    confidence interval
    """
    p_B = p_A * ((100.+lift)/100.)

    interval_covers = 0.0

    for i in range(iters):
        successes_A = binomial(N, p_A)
        observed_failures_A = binomial(N-successes_A, r)
        p_A_posterior = get_p_dist(successes_A, observed_failures_A, r)


        successes_B = binomial(N, p_B)
        observed_failures_B = binomial(N-successes_B, r)
        p_B_posterior = get_p_dist(successes_B, observed_failures_B, r)

        interval = get_lift_confidence_interval(p_A_posterior, p_B_posterior, conf)
        interval_covers += lift_interval_covers_true_lift(interval, lift)
    msg = "%d %% lift interval covers true lift %f%% of the time"
    print msg % (conf, (interval_covers/iters)*100)
~~~
     
The plot below shows what fraction of 90% confidence intervals covered the true lift for the naive method described above. Clearly, the quality of our confidence intervals deteriorates rapidly
as the sampling rate increases. As a sanity check, when we do not sample (\\(r=1\\)) we get correct confidence intervals. But even sampling at rate 1:100 leads to 90% confidence intervals that only
cover the true lift 78% of the time. 

![_config.yml]({{ site.baseurl }}/ipython/AB_testing_with_sampled_data/AB_testing_with_sampled_data_files/AB_testing_with_sampled_data_30_0.png)


The issue with the naive method is that we act as if we have complete certainty in the number
of failed impressions. This assumption becomes more and more inaccurate as we increase the sampling
rate. Lets fix this by incorporating the uncertainty we have in the number of failed
impressions. We will do this by modifying how we sample from the posterior distribution over the donation rates.
Above, we sample repeatedly from a beta with fixed parameters: \\(beta(1+\text{successes}, 1+\text{estimated\_failures})\\). Instead, lets repeatedly take a sample from the distribution over the total number of failures (total_failures_sample) and then sample from a beta with parameters \\(beta(1+{successes}, 1+\text{total\_failures\_sample})\\). In the code below, I sample from the negative binomial instead of the inverse binomial. I do this because I have not yet implemented a generator for the inverse binomial and the negative binomial is a good approximation in most cases. I should also not that in numpy the negative binomial is parameterized in such a way that it returns the number of non-recorded failed impressions before the nth recorded failed impression. To get the total number of failed impressions, we need to add the number of recorded failed impressions to the numpy random variable.



~~~python
def get_p_dist_proper(successes, observed_failures, r):
    missing_failures_distribution =  \
    negative_binomial(observed_failures, r, size = 20000)
    failures_distribution = missing_failures_distribution + observed_failures
    def sample(num_failures):
        return beta(1+successes, 1+num_failures)
    p_hat = map(sample, failures_distribution)
    return np.array(p_hat)
~~~

The plot below shows what fraction of 90% confidence intervals covered the true lift for when we sample from the
posterior distribution over the donation rates as described above.

![_config.yml]({{ site.baseurl }}/ipython/AB_testing_with_sampled_data/AB_testing_with_sampled_data_files/AB_testing_with_sampled_data_32_0.png)

Wonderful! We get 90% confidence intervals that cover the true lift 90% of the time, even as we ramp up the sampling rate. I am embarrassed to say that I cannot provide a proof for why this method works. If you have one, I invite you to link to it in the comments.