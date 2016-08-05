

```python
import numpy as np
from scipy.stats import beta as beta_dist
from random import random
import itertools
from statsmodels.stats.proportion import proportion_confint
from statsmodels.stats.weightstats import ztest
%matplotlib inline
from matplotlib import pyplot as plt
```

# AB Testing and the Importance of Independent Observations

Statistical tests commonly used for AB testing, like the two-sample z-test, rely on the assumption that the experimental observations (i.e. samples) are independent. If this assumption is not met, the test becomes unreliable in the sense that it may not achieve the desired false discovery rate. In this post, we will use simulations to explore how violating the assumption of independent observations can impact the reliability of testing. 

Although there are many ways observations can be become dependent, a common source of dependence between observations in the online setting is multiple observations stemming from the same user. We will see that pooling observations by user and using user-level aggregates as observations removes this type of dependence and leads to reliable testing, when users act independently of each other. This is not a given and should be carefully considered for each use case, however, a discussion of between-user dependence is outside the scope of this post.

We will start by defining a simple model of users on a website interacting with an AB test. Then we will walk through the necessary code to simulate this model and evaluate whether hypothesis testing gives reliable results. Finally, we will see that a naive approach to testing that does not take into account the dependence between a user's actions leads to high false discovery rates and that the dependence can be removed by taking user-level aggregates.

## A simple model of an online AB test

Say we are running a banner on our website with the goal of eliciting some user action, such as making a donation. Assume we show the banner on every pageview until the user makes a donation. When a user $u$ comes to the site, they are first randomly assigned into treatment $A$ or $B$. Depending on the treatment they are in, they are assigned a probability of donating $p\_donate_u$, which is drawn from a beta distribution (either $Beta_A$ or $Beta_B$). On each pageview, $u$ donates with probability $p\_donate_u$. Each user is also assigned a fixed probability of leaving the site after each pageview $p\_exit_u$ which is drawn from the beta distribution $Beta_{exit}$. 

After defining $Beta_A$,  $Beta_B$, $Beta_exit$ and choosing a number of users $N$ to include in the experiment, we can run controlled simulations of AB tests and explore how violations in independence assumptions impact the reliability of the test results.

## Experiment Simulation Code

Let's start by defining a convenience class that lets us draw samples from a beta distribution. Later, we will create instances of this `Beta` class for $Beta_A$, $Beta_B$ and $Beta_{exit}$.


```python
class Beta():
    def __init__(self, a, b):
        self.a = a
        self.b = b
        
    def draw(self):
        return beta_dist.rvs(self.a, self.b)
```

We will use the `User` class to store user specific data including the condition the user is in, their probability of donating and exiting, as well as the a list of results (i.e. donate or not donate) for each banner impression they saw.


```python
class User:
    def __init__(self):
        self.condition = None
        self.p_donate = None
        self.p_exit = None
        self.results = []
```

The `Treatment` class defines how a user moves through an experimental treatment and is defined by user-level distributions over exit and donation probabilities. The `treat` method takes in a user and assigns them a donation and exit probability. Next it simulates a user navigating the website. The user sees an impression on every pageview until they donate or leave the site. The results of each impression are recorded in a user's results attribute. 


```python
class Treatment:
    def __init__(self, condition, beta_donate, beta_exit):
        self.condition = condition
        self.beta_donate = beta_donate
        self.beta_exit = beta_exit
        
    def treat(self, user, max_pagviews=100):
        user.condition = self.condition
        user.p_donate = self.beta_donate.draw()
        user.p_exit = self.beta_exit.draw()
        
        for i in range(max_pagviews):
            if random() < user.p_donate:
                user.results.append(1)
                return user
            else:
                user.results.append(0)      
            if random() < user.p_exit:
                return user
        return user
```

The function `run_experiment` randomly assigns $N$ users to treatments A and B and returns lists of users who went through treatments A and B. The impression results for each of these user groups can be used to perform statistical tests comparing the performance of the two treatments.


```python
def run_experiment(A, B, N):
    users_A = []
    users_B = []
    for i in range(N):
        user = User()
        if random() < 0.5:
            users_A.append(A.treat(user))
        else:
            users_B.append(B.treat(user))
    return users_A, users_B    
```

## Test Evaluation Code

Although there are many ways to do hypothesis testing, we will use the popular two sample ztest. The function `two_sample_ztest` takes a list of users who went through treatment A, a list of users who went through treatment B and a transform function. The transform function maps the list of users into a list of observations (e.g. did a user donate or not for each user). The output is a p-value, which corresponds to the probability of the observed difference in the average observation per treatment under the assumption that the treatments are identical.


```python
def two_sample_ztest(users_A, users_B, transform):
    return ztest(transform(users_A), transform(users_B))[1]
```

If we run many experiments where treatments A and B are identical, then we should observe p-values that are less than $alpha$, in $alpha$*100% of runs . This is what the test is designed to guarantee if its assumptions are met. The function `check_discovery_rate` can be used to check whether this is the case. It takes in 2 treatments, a test and a transform, runs many simulations and computes a 95% confidence interval over the fraction of tests that have a p-value less than $alpha$. 


```python
def check_discovery_rate(A, B, n_samples, test, transform, n_runs=5000, alpha=0.05):
    ps = []
    for i in range(n_runs):
        users_A, users_B = run_experiment(A, B, n_samples)
        ps.append(test(users_A, users_B, transform))
        
    n_significant = (np.array(ps)<alpha).sum()
    return proportion_confint(n_significant, n_runs)
```

##  Comparing Impression and User level conversion rates

The simplest method for comparing treatments is to compute the fraction of impressions that lead to a donation (call this fraction the "impression level conversion rate"). In this case our observations are the results of each impression (i.e. donation or no donation). The transform function `get_impression_results` takes in a list of users and concatenates the impression-level results per user into one big vector. Taking the average of the elements of this vector gives the empirical probability that an impression leads to a donation.


```python
def get_impression_results(users):
    data = [u.results for u in users]
    return list(itertools.chain.from_iterable(data))
```

We will simulate thousands of experiments using this transform and check the discovery rate of our hypothesis test. The code below defines two treatments $A$ and $B$, that are in fact identical. 


```python
beta_donate_a = Beta(10, 10)
beta_donate_b = beta_donate_a
beta_exit = Beta(2, 10)

A = Treatment('A', beta_donate_a, beta_exit)
B = Treatment('B', beta_donate_b, beta_exit)
```

Now we can use `check_discovery_rate` to see how often we get a false discovery (i.e. the test declares that one treatment is different from the other). We will run our test at significance level alpha = 0.05. Hence, we should expect 5% of experiments to lead to a false discovery.


```python
check_discovery_rate(A, B, 100, two_sample_ztest, get_impression_results, alpha=0.05)
```




    (0.066744214388100423, 0.08125578561189957)



The expected false discovery rate of 0.05 is not in the interval, indicating that there is a serious flaw in the testing methodology. The fact that the observed false discovery rate is higher than expected means that the test is overconfident. It underestimates the probability of the differences in the average observation between treatments. 

The problem, of course, is that the sample contains multiple impressions for at least some users and impression results from the same user are not independent. To see why, consider an extreme example were a user will either donate on their first impression or not at all. Clearly, the impression results from users who do not donate are dependent: if we know that one of their impressions did not lead to a donation, then we know that none of the others will either.

The simply remedy to our dependence problem is to move from impression-level observations to user-level observations. The transform function `get_user_results` takes in a list of users and determines whether each user made a donation across all their impressions. It returns a vector with an element for each user. The element is 1 if the user made a donation, 0 otherwise. Taking the average of the elements of this vector gives the empirical probability that an user makes a donation. 


```python
def get_user_results(users):
    return [sum(u.results) for u in users]
```

Again,  we can use `check_discovery_rate` to see how often we get a false discovery.


```python
check_discovery_rate(A, B, 100, two_sample_ztest, get_user_results)
```




    (0.040757568610113086, 0.052442431389886919)



Now the expected false discovery rate of 0.05 sits squarely in the interval, as we would expect (to happen in 95% of simulations) when we correctly apply the hypothesis test using independent observations.

Note that users act independently by design of the simulation. This may not always be the case. You need to evaluate whether this is a good assumption for every use case. Think hard about the question "Is there a way one user's actions can effect another user's actions?". In our simple banner scenario the answer is likely no. But this can quickly change. For example, if the banners showed a live progress bar of the total number of donations received, then users might change their behavior based on the behavior of others. Another source of between user dependence could arise, if after donating, users where encouraged to brag about their donation on a social network like Facebook or Twitter. Again, a discussion of the myriad was to address between-user dependence is beyond the scope of this post. The main take away is that you need to think carefully about dependence between observations, whenever you are designing a test and that when running tests online, you will almost always need to take user level aggregates.


```python

```
