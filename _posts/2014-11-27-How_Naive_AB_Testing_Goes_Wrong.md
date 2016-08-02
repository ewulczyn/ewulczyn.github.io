---
layout: post
title: How Naive AB Testing Goes Wrong and How to Fix It
---

Although classical hypothesis tests are probably still the most
widespread technique for online A/B tests, they are not the
most appropriate. In this post, I will describe the shortcomings of hypothesis testing for web optimization and propose a method of repeated Bayesian A/B testing to address them.

To make the discussion more concrete, imagine we are running a website that depends on user donations. For one month a year, we run fundraising banners on our site and our goal is to maximize the number of donations we receive. Our designers have two different ideas for banner designs and we want to choose the one that maximizes the probability that a user will make a donation by their 10th banner impression (call this probability the conversion rate). The current best practice is to run an online randomized controlled experiment and evaluate the results using a classical hypothesis test, like the two sample z-test. The issue is that these tests are not a great fit for this use-case.

Hypothesis testing was designed for situations in which:

**1) the goal is to control the probability of declaring a difference in
conversion rates when there is none**


Most academic scientists care about this because it protects from false
discoveries, which are embarrassing at best. But in the banner optimization setting and most other web optimization settings, we actually don't care about declaring the banners different when they are not. When the banners are equivalent in terms of conversion rates (CR),
there is no bad decision. In the worst case, it may set the designers down the wrong path, because they think one of their ideas is better and that changes how they iterate on their designs. The
choice, however, has no impact on the expected revenue. What we really want, is
to control *the probability of declaring the worse banner the winner when the
banners are in fact different.*


**2) the sample size is predetermined or fixed in advance**


When running web experiments, most of the time you can choose how long to run an experiment. The longer the experiment runs, the more users are added to the experiment and the more information you have to estimate which banner is better. Hypothesis testing does not take advantage of this fact.
Instead, the best practice is to calculate the sample size (in our case the number of users in the experiment) in advance. To determine the sample size, you need to have an estimate of the CR
of your control banner and decide on the minimum effect size you want to detect
(I will use MDE as an abbreviation for the minimum detectable effect size going forward). You also need to choose the desired percent of the time a difference will
be detected, assuming one does not exist (significance or alpha) and the percent of
the time the MDE will be detected (power or beta). For more theoretical background on sample size determination, check out the [Wikipedia page](http://en.wikipedia.org/wiki/Sample_size_determination). For an [online calculator](http://www.evanmiller.org/ab-testing/sample-size.html), see Evan
Miller's blog.

Holding all other parameters equal, the sample size increases as you decrease the MDE. In other words, the larger the true difference, the less data you need for significance. Now imagine a scenario in which you decided you want to be able to detect a 5% difference in
CRs. If the new banner is actually 50% better than the control,
you have set yourself up for losing a lot of revenue on the portion of users
exposed to the control. Instead of committing to a sample size in advance, we
would really like our testing procedure to have *an adaptive sample size*. The
larger the true difference in CRs, the shorter it should run. This should be
possible since larger differences are easier detect with less data.

A common mistake is to run multiple null
hypothesis tests as the data are coming in and decide to stop the test early on the first significant result. There
can be many valid reasons for peeking at test results that trump maintaining
statistical soundness. One example, is to check that results are logged properly or that the banners are appearing on all browsers. The problem with repeated hypothesis testing, however, is that it corrupts the test. The more often you peek the higher the effective probability of declaring a difference in CRs when there is none. You also decrease the probability of deciding that the better banner is the winner, which is what we actually care about. The advantage, of course is that you might stop the test early and get away with a smaller sample size.


To summarize, we need a testing procedure that:

**1. requires fewer samples as the difference in CRs increases,
while**

**2. controlling the probability of declaring the worse banner the winner**


What follows is the development of a method that meets these ends. It is based on simulation and the repeated evaluation of a Bayesian stopping criterion proposed by [Chris Stucchio](https://www.chrisstucchio.com/blog/2014/bayesian_ab_decision_rule.html)


## Investigating the Effects of Repeated Testing

Before taking the leap from frequentist to Bayesian statistics, I want to explore the pros and cons of repeated hypothesis testing in a formal setting. Consider the following repeated testing procedure:

~~~
while true:
    collect results from n new users per banner
    evaluate a ttest on the difference in CRs 
    if the difference is significant:
        stop the test
        declare the banner with the higher CR the winner
    if you hit the classically determined sample size:
        stop the test
        declare the winner unknown
~~~
To paraphrase, we evaluate a hypothesis test every time we get n new samples (per banner) until we
reach significance or hit the fixed predetermined sample size. If we stopped
because we hit significance, we declare the banner with the higher CR at
termination to be the winner. If we stopped because we hit the max sample
size, we declare the winner to be unknown. We can answer the following questions through running simulations:

**1. How does the probability of picking the better banner change as a function of the difference in CRs?**

**2. How does the expected sample size change as a function of the difference in CRs?**

**3. How does changing the testing frequency effect the above**


We will simulate donations from users in group B by sampling from a Bernoulli
distribution Bernoulli(CR\_B). We can emulate exposing users to a synthetic
banner A with y percent lift over banner B, by sampling from Bernoulli(CR\_B *
((100+y)/100)). For a given percent lift y, we can run the procedure described
above k times to get an estimate for the probability of declaring A the winner
and the distribution over the number of samples needed to terminate.

As an example, lets set the true CR of the control banner B to be 0.2. In
practice, we will not know the true value of CR\_B. We will need to estimate
it either from historical data or by running it by itself for a while. 
(The statistics underlying most behavior on the web are not stationary, so beware
that the data used for your estimate is reflective of the conditions under which
you will actually test the banners). One concern might be that the simulated results are quite sensitive to errors in
estimation. To account for this we can form a confidence interval around CR\_B
and use the lower and upper bounds of the confidence interval to anchor the
simulation and see how the results differ. If the difference is too large, we
will need to get more data on Banner B to form a tighter interval.

For the following experiments I simulated donation results from 5000 users using a banner with CR = 0.2
and formed a 95% credible interval:

~~~ ruby
import numpy as np
from numpy.random import beta, binomial

n_users = 5000
n_donations = binomial(n_users, 0.2)
CR_B_distribution = beta(n_donations, n_users-n_donations, 10000)

print 'Lower Bound %0.3f' % np.percentile(CR_B_distribution, 2.5)
print 'Upper Bound %0.3f' % np.percentile(CR_B_distribution, 97.5)


Lower Bound 0.192
Upper Bound 0.214
~~~


Lets set MDE = 0.05, beta = 0.95, alpha = 0.05 and do a hypothesis test every time we get the results from n = 250 new users. For the figures below, I ran the simulation 1000 times. The first plot shows the probability of deciding that banner A is the winner as a function of the percent lift A has over B. The second plot shows the expected number of samples needed per banner to stop the test, where the gray area indicates a 90% credible interval. 


![_config.yml]({{ site.baseurl }}/ipython/how_naive_ab_testing_goes_wrong/how_naive_ab_testing_goes_wrong_files/how_naive_ab_testing_goes_wrong_3_1.png)



The first thing to notice is that even though the confidence interval for CR\_B is pretty loose, there is little difference between using the upper or lower bound. Second, the probability of declaring a winner when the lift is 0 is 0.39, even though the single null
hypothesis tests where testing at significance 0.05. Here you have the
corrupting power of peeking at p_vlaues in action! But this algorithm has its
redeeming qualities. Look at the expected number of samples needed as a function
of the percent lift in CR. We can detect a 10% lift with probability 0.988 with
only an average of 3200 samples, less than one tenth of the classically determined sample size (which is 41950)! To get a deeper understanding of the effect of changing the test interval, lets try the extremes: testing only once and testing every time a new sample comes in. 

Here are the results when testing only once after the classically computed number of samples.

![_config.yml]({{ site.baseurl }}/ipython/how_naive_ab_testing_goes_wrong/how_naive_ab_testing_goes_wrong_files/how_naive_ab_testing_goes_wrong_4_1.png)


We see that when the lift is 0, there is roughly a 5% chance of declaring a
winner. This false discovery rate corresponds to the significance level of the
hypothesis test, which we set to 5%. We also specified that we want to be
able to detect a 5% effect 95% of the time. In the simulation, we detected a 5%
gain over the control CR 94.5% of the time and 5% drop 95.2% of the time. The
results of the simulation match the design of the hypothesis test as
expected. Here is what happens when we go to the other extreme and test after every new pair of samples:

![_config.yml]({{ site.baseurl }}/ipython/how_naive_ab_testing_goes_wrong/how_naive_ab_testing_goes_wrong_files/how_naive_ab_testing_goes_wrong_5_1.png)

Now, we make false discoveries 66% of the time. We declare a banner with a
5% lift over the control as the winner only 81.5% of the time and and banner
with a %-5% lift as the winner 16.7% of the time. The advantage of testing after every
new pair of samples is that the expected sample size of the test is minimized. The takeaway is
that you can trade off accuracy with sample size by changing the interval at
which you test.

Notice that there is something strange going on when we run the simulation at
0 percent lift. We are waiting around for the hypothesis to falsely declare
significance. Intuitively, we are running through samples waiting for something unlikely to happen. It would be better to terminate the experiment when we are reasonably sure that the banners where the same and that the expected cost
of choosing one over the other is small. The next section will try to address this shortcoming. 

## Repeated Testing with a Bayesian Stopping Criterion


The procedure I will describe next also uses repeated testing, but we will swap out the
null hypothesis test with a different stopping criterion. Instead of stopping
when the probability of the observed difference in CRs under the assumption
that the banners have the same CR is below our significance level alpha, lets stop when the expected cost associated with our decision is
low. This idea comes from Chris Stuchio's [recent
post](http://www.bayesianwitch.com/blog/2014/bayesian_ab_test.html).


To achieve this objective, we will need to switch to the Bayesian formalism, in
which we treat our conversion rates not as unknown fixed parameters, but as
random variables. For a primer on Bayesian AB testing, I recommend Sergey
Feldman's [blog post](http://engineering.richrelevance.com/bayesian-ab-tests/).
Using the techniques described in Feldman's post, we can form a numeric
representation of the joint posterior distribution over the conversion rates
CR\_A and CR\_B, which will allow us to get distributions over functions of the CRs.

Say that after collecting N samples, you find banner A has a higher empirical
conversion rate than banner B. To decide whether to stop the test, we could see how much of a chance remains
that B is better that A and stop if P(CR\_B > CR\_A) is below a threshold. The
rub is that if the banners really where the same, then P(CR\_B < CR\_A) should
be 0.5. We are in the same situation as before,  where we are waiting for a false result to terminate our test. To avoid this, consider the following cost function (which is itself a random variable):

~~~
f(CR_A, CR_B) =        0         if CR_A > CR_B
                  (CR_B - CR_A)  if CR_B > CR_A
~~~

We incur 0 cost if CR_A is greater then CR_B and incur the difference in
CRs otherwise. We can run the test until the expected
cost E[f(CR\_A, CR\_b)] drops below a threshold. The code snippet below shows how to compute the expected cost through numerical integration, given the number of donations and users for both banners:

```python
import numpy as np
from numpy.random import beta

def expected_cost(a_donations, a_users, b_donations, b_users, num_samples=10000):
    #form the posterior distribution over the CRs
    a_dist = beta(1 + a_donations, 1 + a_users - a_donations, num_samples) 
    b_dist = beta(1 + b_donations, 1 + b_users - b_donations, num_samples)
    #form the distribution over the cost
    cost_dist = np.maximum((b_dist-a_dist), 0) 
    # return the expected cost
    return cost_dist.mean()
```

    
Chris Stucchio derived a [closed form solution](https://www.chrisstucchio.com/blog/2014/bayesian_ab_decision_rule.html) for this cost function. This is
certainly more computationally efficient, but the advantage of the numeric
integration is that it is easy to experiment with slight modifications to the cost
function. For example you could experiment with incurring the relative cost
(CR\_B - CR\_A)/ CR\_A instead of the absolute cost CR\_B - CR\_A.

Below are the results for computing the expected cost every 250 samples and
stopping if the expected cost drops below 0.0002. Picking a cost threshold that allows for a comparison with using hypothesis tests as a stopping criterion, is difficult. The comparison will not be straightforward, but this cost threshold gets us into the right ball park.


![_config.yml]({{ site.baseurl }}/ipython/how_naive_ab_testing_goes_wrong/how_naive_ab_testing_goes_wrong_files/how_naive_ab_testing_goes_wrong_6_1.png)

A nice feature is that this method only requires 2 hyper-parameters, the cost threshold and the interval at which to evaluate the stopping criterion. Since we are not waiting for a statistical fluke to end the test at 0% lift, we don't need to set a maximum sample size. Hence, we always declare a winner. Of course, we can set a finite sample size if we need to. This will result in a non-zero chance of declaring the winner to be unknown and decrease the expected sample size, especially when the lift is small in magnitude. 


The high variance in the number of samples needed prior to termination around 0 lift, makes for a difficult comparison with the hypothesis test stopping criterion, so here is a table comparing the results from the hypothesis stopping criterion and the expected cost stopping criterion, with a maximum sample size per banner of 41950.



<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(A wins) NHT</th>
      <th>P(A wins) Cost</th>
      <th>P(Unknown) NHT</th>
      <th>P(Unknown) Cost</th>
      <th>Avg Time NHT</th>
      <th>Avg Time Cost</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-20.0</td>
      <td> 0.001</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td>   916</td>
      <td>   910</td>

    </tr>
    <tr>
      <td>-10.0</td>
      <td> 0.005</td>
      <td> 0.005</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td>  3083</td>
      <td>  2390</td>

    </tr>
    <tr>
      <td> -5.0</td>
      <td> 0.030</td>
      <td> 0.041</td>
      <td> 0.021</td>
      <td> 0.000</td>
      <td> 10161</td>
      <td>  6098</td>

    </tr>
    <tr>
      <td> -2.5</td>
      <td> 0.061</td>
      <td> 0.110</td>
      <td> 0.288</td>
      <td> 0.065</td>
      <td> 20683</td>
      <td> 12282</td>

    </tr>
    <tr>
      <td> -1.5</td>
      <td> 0.114</td>
      <td> 0.159</td>
      <td> 0.453</td>
      <td> 0.137</td>
      <td> 24557</td>
      <td> 14828</td>
    </tr>
    <tr>
      <td>  0.0</td>
      <td> 0.195</td>
      <td> 0.408</td>
      <td> 0.611</td>
      <td> 0.220</td>
      <td> 28529</td>
      <td> 17409</td>
    </tr>
    <tr>
      <td>  1.5</td>
      <td> 0.420</td>
      <td> 0.683</td>
      <td> 0.473</td>
      <td> 0.125</td>
      <td> 25655</td>
      <td> 14700</td>
    </tr>
    <tr>
      <td>  2.5</td>
      <td> 0.635</td>
      <td> 0.814</td>
      <td> 0.277</td>
      <td> 0.072</td>
      <td> 20099</td>
      <td> 12815</td>

    </tr>
    <tr>
      <td>  5.0</td>
      <td> 0.938</td>
      <td> 0.964</td>
      <td> 0.021</td>
      <td> 0.001</td>
      <td> 10065</td>
      <td>  6305</td>

    </tr>
    <tr>
      <td> 10.0</td>
      <td> 0.990</td>
      <td> 0.993</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td>  3163</td>
      <td>  2521</td>


    </tr>
    <tr>
      <td> 20.0</td>
      <td> 0.998</td>
      <td> 1.000</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td>  1027</td>
      <td>  1017</td>

    </tr>
  </tbody>
</table>
</div>

For the particular parameters I choose, the cost criterion has a uniformly higher chance of declaring A the better banner, when the lift is positive.
When the lift is negative it also has a higher chance of declaring A the winner. This is largely because it declares the winner to be unknown much less often and so is more likely to make mistakes. The big win of the cost method comes from needing only 40% of the samples when the lift is less than 20%. The goal of the comparison is more to characterize the methods than to declare a winner. I suggest playing around with both.


## Bayesian Tests are not Immune to Peeking

Before signing off, I want illustrate the effects of peeking at Bayesian A/B test results. The more often you peek, the more often you decide the worse banner is the winner if you set yourself a stopping criterion. In this respect, the Bayesian A/B test behaves similarly to the hypothesis test. The plot below shows the probability of A winning and the expected sample size as a function of lift for different testing intervals. To generate the plots, I increased the cost threshold to 0.001 and set the maximum sample size to 40000. We see the same pattern as in repeated hypothesis testing: you can trade off accuracy with sample size by changing the interval at which you test. 

![_config.yml]({{ site.baseurl }}/ipython/how_naive_ab_testing_goes_wrong/how_naive_ab_testing_goes_wrong_files/how_naive_ab_testing_goes_wrong_7_1.png)