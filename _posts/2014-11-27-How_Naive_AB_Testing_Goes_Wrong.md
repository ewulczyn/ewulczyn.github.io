---
layout: post
title: How Naive AB Testing Goes Wrong and How to Fix It
---

At the Wikimedia Foundation we use A/B testing to optimize the number of
donations per impression from our fundraising banners. Although classical hypothesis tests are probably still the most
widespread technique for online A/B tests, they are not the
most appropriate. In this post, I will describe the shortcomings of hypothesis testing for web optimization and propose a method of repeated Bayesian A/B testing to address them.

 Hypothesis testing was designed for situations in which:


**1) the goal is to control the probability of declaring a difference in
population means when there is none**


Most academic scientists care about this because it protects from false
discoveries, which are embarrassing at best. But in the banner optimization setting and most other web optimization settings, we actually don't care about declaring the variants different when they are not. When the banners are equivalent in terms of click through rates (CTR),
there is no bad decision. In the worst case, it may set the designers down the wrong path, because they think a change they made to the control banner is a big win. The
choice, however, has no impact on the expected revenue. What we really want, is
to control *the probability of declaring the worse banner the winner when the
banners are in fact different.*



**2) the sample size is predetermined or fixed in advance**


When running web experiments, most of the time you can choose how long to run an experiment. Hypothesis testing does not take advantage of this fact.
Instead, the best practice is to calculate the sample size in advance of running
the test. To determine the sample size, you need to have an estimate of the CTR
of your control banner and decide on the minimum effect size you want to detect
(I will use MDE as an abbreviation for the minimum detectable effect size going forward). You also need to choose the desired percent of the time a difference will
be detected, assuming one does not exist (significance or alpha) and the percent of
the time the MDE will be detected (power or beta). For more theoretical background on sample size determination, check out the [Wikipedia page](http://en.wikipedia.org/wiki/Sample_size_determination). For an [online calculator](http://www.evanmiller.org/ab-testing/sample-size.html), see Evan
Miller's blog.

Holding all other parameters equal, the sample size increases as you decrease the MDE. In other words, the larger the true difference, the less data you need for significance. Now imagine a scenario in which you decided you want to be able to detect a 5% difference in
click through rates. If the new banner is actually 50% better than the control,
you have set yourself up for losing a lot of revenue on the portion of traffic
exposed to the the control. Instead of committing to a sample size in advance, we
would really like our testing procedure to have *an adaptive sample size*. The
larger the true difference in CTRs, the shorter it should run. This should be
possible since larger differences are easier detect with less data.

A common mistake is to run multiple null
hypothesis tests as the data are coming in and decide to stop the test early on the the first significant result. There
can be many valid reasons for peeking at test results that trump maintaining
statistical soundness. One example, is to check for glitches in the testing
framework. The problem with repeated hypothesis testing, however, is that it corrupts the test. The more often you peek the higher the effective probability of declaring a difference in CTRs when there is none. You also decrease the probability deciding that the better banner is the winner, which is what we actually care about. The advantage, of course is that you might stop the test early and get away with a smaller sample size.


To summarize, we need a testing procedure that:

**1. requires fewer samples as the difference in CTRs increases,
while**

**2. controlling the probability of declaring the worse banner the winner**


What follows is the development of a method that meets these ends. It is based on simulation and the repeated evaluation of a Bayesian stopping criterion proposed by [Chris Stucchio](https://www.chrisstucchio.com/blog/2014/bayesian_ab_decision_rule.html)


##Investigating the Effects of Repeated Testing. 

Before taking the leap from frequentist to Bayesian statistics, I want to explore the pros and cons of repeated hypothesis testing in a formal setting. Consider the following repeated testing procedure:

~~~
while true:
    run each banner for n impressions
    evaluate a ttest on the difference in CTRs 
    if the difference is significant:
        stop the test
        declare the banner with the higher CTR the winner
    if you hit the classically determined sample size:
        stop the test
        declare the winner unknown
~~~
To paraphrase, we evaluate a hypothesis test every n records until we
reach significance or hit the predetermined sample size. If we stopped
because we hit significance, we declare the banner with the higher CTR at
termination to be the winner. If we stopped because we hit the max sample
size, we declare the winner to be unknown. We can answer the following questions through running simulations:

**1. How does the probability of picking the better banner change as a function of the difference in CTRs?**

**2. How does the expected run time change as a function of the difference in CTRs?**

**3. How does changing the testing frequency effect the above**


We will simulate impressions of banner B by sampling from a Bernoulli
distribution Bernoulli(CTR\_B). We can emulate impressions from synthetic
banner A with y percent lift over banner B, by sampling from Bernoulli(CTR\_B *
((100+y)/100)). For a given percent lift y, we can run the procedure described
above k times to get an estimate for the probability of declaring A the winner
and the distribution over the number of samples needed to terminate.

As an example, lets set the true CTR of the control banner B to be 0.2. In
practice, we will not know the true value of CTR\_B. We will need to estimate
it either from historical data or by running it by itself for a while. 
(The statistics underlying most behavior on the web are not stationary, so beware
that the data used for your estimate is reflective of the conditions under which
you will actually test the banners). One concern might be that the simulated results are quite sensitive to errors in
estimation. To account for this we can form a confidence interval around CTR\_B
and use the lower and upper bounds of the confidence interval to anchor the
simulation and see how the results differ. If the difference is too large, we
will need to get more data on Banner B to form a tighter interval.

 For the following experiments I simulated 5000 impressions of a banner with CTR = 0.2
and formed a 95% credible interval:

~~~ ruby
import numpy as np
from numpy.random import beta, binomial

impressions = 5000
clicks = binomial(impressions, 0.2)
CTR_B_distribution = beta(clicks, impressions-clicks, 10000)

print 'Lower Bound %0.3f' % np.percentile(CTR_B_distribution, 2.5)
print 'Upper Bound %0.3f' % np.percentile(CTR_B_distribution, 97.5)


Lower Bound 0.192
Upper Bound 0.214
~~~


Lets set MDE = 0.05, beta = 0.95, alpha = 0.05 and do a hypothesis test every n = 250 records. For the figures below, I ran the simulation 1000 times. The first plot shows the probability of deciding that banner A is the winner as a function of the percent lift A has over B. The second plot shows the expected number of samples needed per banner to stop the test, where the gray area indicates a 90% credible interval. 


![_config.yml]({{ site.baseurl }}/images/2014-11-27/Self-Terminating AB Test-Blog_3_1.png)



<!---

<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(Choosing A) Median</th>
      <th>P(Unknown) Median</th>
      <th>Avg Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-20.0</td>
      <td> 0.001</td>
      <td> 0.000</td>
      <td>   916</td>
    </tr>
    <tr>
      <td>-10.0</td>
      <td> 0.005</td>
      <td> 0.000</td>
      <td>  3083</td>
    </tr>
    <tr>
      <td> -5.0</td>
      <td> 0.030</td>
      <td> 0.021</td>
      <td> 10161</td>
    </tr>
    <tr>
      <td> -2.5</td>
      <td> 0.061</td>
      <td> 0.288</td>
      <td> 20683</td>
    </tr>
    <tr>
      <td> -1.5</td>
      <td> 0.114</td>
      <td> 0.453</td>
      <td> 24557</td>
    </tr>
    <tr>
      <td>  0.0</td>
      <td> 0.195</td>
      <td> 0.611</td>
      <td> 28529</td>
    </tr>
    <tr>
      <td>  1.5</td>
      <td> 0.420</td>
      <td> 0.473</td>
      <td> 25655</td>
    </tr>
    <tr>
      <td>  2.5</td>
      <td> 0.635</td>
      <td> 0.277</td>
      <td> 20099</td>
    </tr>
    <tr>
      <td>  5.0</td>
      <td> 0.938</td>
      <td> 0.021</td>
      <td> 10065</td>
    </tr>
    <tr>
      <td> 10.0</td>
      <td> 0.990</td>
      <td> 0.000</td>
      <td>  3163</td>
    </tr>
    <tr>
      <td> 20.0</td>
      <td> 0.998</td>
      <td> 0.000</td>
      <td>  1027</td>
    </tr>
  </tbody>
</table>
</div>

-->


The first thing to notice is that even though the confidence interval for CTR\_B is pretty loose, there is little difference between using the upper or lower bound. Second, the probability of declaring a winner when the lift is 0 is 0.39, even though the single null
hypothesis tests where testing at significance 0.05. Here you have the
corrupting power of peeking at p_vlaues in action! But this algorithm has its
redeeming qualities. Look at the expected number of samples needed as a function
of the percent lift in CTR. We can detect a 10% lift with probability 0.988 with
only an average of 3200 records, less than one tenth of the classically determined sample size (which is 41950)!


To get a deeper understanding of what the effect changing the test interval, lets try the extremes: testing at every new pair of records and testing only once. Here are the results when testing only once and after the classically computed number of records.

![_config.yml]({{ site.baseurl }}/images/2014-11-27/Self-Terminating AB Test-Blog_7_1.png)

<!--
<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(Choosing A) Median</th>
      <th>P(Unknown) Median</th>
      <th>Avg Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-20.0</td>
      <td> 0.0000</td>
      <td> 0.0000</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td>-10.0</td>
      <td> 0.0000</td>
      <td> 0.0000</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td> -5.0</td>
      <td> 0.0000</td>
      <td> 0.0476</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td> -2.5</td>
      <td> 0.0000</td>
      <td> 0.5536</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td> -1.5</td>
      <td> 0.0016</td>
      <td> 0.8042</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td>  0.0</td>
      <td> 0.0272</td>
      <td> 0.9494</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td>  1.5</td>
      <td> 0.1878</td>
      <td> 0.8106</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td>  2.5</td>
      <td> 0.4264</td>
      <td> 0.5734</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td>  5.0</td>
      <td> 0.9464</td>
      <td> 0.0536</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td> 10.0</td>
      <td> 1.0000</td>
      <td> 0.0000</td>
      <td> 41951</td>
    </tr>
    <tr>
      <td> 20.0</td>
      <td> 1.0000</td>
      <td> 0.0000</td>
      <td> 41951</td>
    </tr>
  </tbody>
</table>
</div>

-->

We see that when the lift is 0, there is roughly a 5% chance of declaring a
winner. This false discovery rate corresponds to the significance level of the
hypothesis test, which we set to 5%. We also specified that we want to be
able to detect a 5% effect 95% of the time. In the simulation, we detected a 5%
gain over the control CTR 94.5% of the time and 5% drop 95.2% of the time. The
results of the simulation match the design of the hypothesis test as
expected. Here is what happens when we go to the other extreme and test after every pair of records:

![_config.yml]({{ site.baseurl }}/images/2014-11-27/Self-Terminating AB Test-Blog_9_1.png)

<!---
<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(Choosing A)</th>
      <th>P(Unknown)</th>
      <th>Avg Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-0.200</td>
      <td> 0.070</td>
      <td> 0.000</td>
      <td>   518</td>
    </tr>
    <tr>
      <td>-0.100</td>
      <td> 0.115</td>
      <td> 0.000</td>
      <td>  1807</td>
    </tr>
    <tr>
      <td>-0.050</td>
      <td> 0.169</td>
      <td> 0.016</td>
      <td>  5645</td>
    </tr>
    <tr>
      <td>-0.025</td>
      <td> 0.225</td>
      <td> 0.174</td>
      <td> 12264</td>
    </tr>
    <tr>
      <td>-0.015</td>
      <td> 0.240</td>
      <td> 0.251</td>
      <td> 13660</td>
    </tr>
    <tr>
      <td> 0.000</td>
      <td> 0.326</td>
      <td> 0.329</td>
      <td> 15751</td>
    </tr>
    <tr>
      <td> 0.015</td>
      <td> 0.486</td>
      <td> 0.279</td>
      <td> 15103</td>
    </tr>
    <tr>
      <td> 0.025</td>
      <td> 0.593</td>
      <td> 0.196</td>
      <td> 12534</td>
    </tr>
    <tr>
      <td> 0.050</td>
      <td> 0.815</td>
      <td> 0.011</td>
      <td>  5628</td>
    </tr>
    <tr>
      <td> 0.100</td>
      <td> 0.860</td>
      <td> 0.000</td>
      <td>  1949</td>
    </tr>
    <tr>
      <td> 0.200</td>
      <td> 0.934</td>
      <td> 0.000</td>
      <td>   607</td>
    </tr>
  </tbody>
</table>
</div>


-->

Now, we make false discoveries 66% of the time. We declare a banner with a
5% lift over the control as the winner only 81.5% of the time and and banner
with a %-5% lift as the winner 16.7% of the time. The advantage of testing every
record is that the expected run time of the test is minimized. The takeaway is
that you can trade off accuracy with sample size by changing the interval at
which you test.

There is something strange going on when we run the simulation at
0 percent lift. We are waiting around for the hypothesis to falsely declare
significance. Intuitively, we are running through samples waiting for something unlikely to happen. It would be better to terminate the experiment when we are reasonably sure that the banners where the same and that the expected cost
of choosing one over the other is small. The next section will try to address this shortcoming. 

##Repeated Testing with a Bayesian Stopping Criterion. 


The procedure I will describe next also uses repeated testing, but we will swap out the
null hypothesis test with a different stopping criterion. Instead of stopping
when the probability of the observed difference in means under the assumption
that the banners have the same CTR is below our significance level alpha, lets stop when the expected cost associated with our decision is
low. This idea comes from Chris Stuchio's [recent
post](http://www.bayesianwitch.com/blog/2014/bayesian_ab_test.html).


To achieve this objective, we will need to switch to the Bayesian formalism, in
which we treat our click through rates not as unknown fixed parameters, but as
random variables. For a primer on Bayesian AB testing, I recommend Sergey
Feldman's [blog post](http://engineering.richrelevance.com/bayesian-ab-tests/).
Using the techniques described in Feldman's post, we can form a numeric
representation of the joint posterior distribution over the click through rates
CTR\_A and CTR\_B, which will allow us to get distributions over functions of the CTRs.

Say that after collecting N samples, you find banner A has a higher empirical
click through rate than banner B. To decide whether to stop the test, we could see how much of a chance remains
that B is better that A and stop if P(CTR\_B > CTR\_A) is below a threshold. The
rub is that if the banners really where the same, then P(CTR\_B < CTR\_A) should
be 0.5. We are in the same situation as before,  where we are waiting for a false result to terminate our test. To avoid this, consider the following cost function (which is itself a random variable):

~~~
f(CTR_A, CTR_B) =        0         if CTR_A > CTR_B
                  (CTR_B - CTR_A)  if CTR_B > CTR_A
~~~

We incur 0 cost if CTR_A is greater then CTR_B and incur the difference in
CTRs otherwise. We can run the test until the expected
cost E[f(CTR\_A, CTR\_b)] drops below a threshold. The code snippet below shows how to compute the expected cost through numerical integration, given the number of clicks and impressions for both banners:

```python
import numpy as np
from numpy.random import beta

def expected_cost(a_clicks, a_impressions, b_clicks, b_impressions, num_samples=10000):
    #form the posterior distribution over the CTRs
    a_dist = beta(1 + a_clicks, 1 + a_impressions - a_clicks, num_samples) 
    b_dist = beta(1 + b_clicks, 1 + b_impressions - b_clicks, num_samples)
    #form the distribution over the cost
    cost_dist = np.maximum((b_dist-a_dist), 0) 
    # return the expected cost
    return cost_dist.mean()
```

    
Chris Stucchio derived a [closed form solution](https://www.chrisstucchio.com/blog/2014/bayesian_ab_decision_rule.html) for this cost function. This is
certainly more computationally efficient, but the advantage of the numeric
integration is that it is easy to experiment with slight modifications to the cost
function. For example you could experiment with incurring the relative cost
(CTR\_B - CTR\_A)/ CTR\_A instead of the absolute cost CTR\_B - CTR\_A.

Below are the results for computing the expected cost every 250 records and
stopping if the expected cost drops below 0.0002. Picking a cost threshold that allows for a comparison with using hypothesis tests as a stopping criterion, is difficult. the comparison will not be straightforward, but this cost threshold gets us into the right ball park.


![_config.yml]({{ site.baseurl }}/images/2014-11-27/Self-Terminating AB Test-Blog_19_1.png)

<!--
<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(Choosing A) Median</th>
      <th>P(Unknown) Median</th>
      <th>Avg Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-20.0</td>
      <td> 0.000</td>
      <td> 0</td>
      <td>   880</td>
    </tr>
    <tr>
      <td>-10.0</td>
      <td> 0.007</td>
      <td> 0</td>
      <td>  2366</td>
    </tr>
    <tr>
      <td> -5.0</td>
      <td> 0.025</td>
      <td> 0</td>
      <td>  6009</td>
    </tr>
    <tr>
      <td> -2.5</td>
      <td> 0.121</td>
      <td> 0</td>
      <td> 13803</td>
    </tr>
    <tr>
      <td> -1.5</td>
      <td> 0.172</td>
      <td> 0</td>
      <td> 20308</td>
    </tr>
    <tr>
      <td>  0.0</td>
      <td> 0.471</td>
      <td> 0</td>
      <td> 32284</td>
    </tr>
    <tr>
      <td>  1.5</td>
      <td> 0.803</td>
      <td> 0</td>
      <td> 20309</td>
    </tr>
    <tr>
      <td>  2.5</td>
      <td> 0.897</td>
      <td> 0</td>
      <td> 13176</td>
    </tr>
    <tr>
      <td>  5.0</td>
      <td> 0.959</td>
      <td> 0</td>
      <td>  6376</td>
    </tr>
    <tr>
      <td> 10.0</td>
      <td> 0.991</td>
      <td> 0</td>
      <td>  2602</td>
    </tr>
    <tr>
      <td> 20.0</td>
      <td> 1.000</td>
      <td> 0</td>
      <td>  1063</td>
    </tr>
  </tbody>
</table>
</div>
-->

A nice feature is that this method only requires 2 hyper-parameters, the cost threshold and the interval at which to evaluate the stopping criterion. Since we are not waiting for a statistical fluke to end the test at 0% lift, we don't need to set a maximum run time. Hence, we always declare a winner. Of course, we can set a finite run time if we need to. This will result in a non-zero chance of declaring the winner to be unknown and decrease the expected run time, especially when the lift is small in magnitude. 


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


##Bayesian Tests are not Immune to Peeking

Before signing off, I want illustrate the effects of peeking at Bayesian A\B test results. The more often you peek, the more often you decide the worse banner is the winner if you set yourself a stopping criterion. In this respect, the Bayesian A/B test behaves similarly to the hypothesis test. The plot below shows the probability of A winning and the expected run time as a function of lift for different testing intervals. To generate the plots, I lowered the cost threshold to 0.001 and set the maximum sample size to 40000. We see the same pattern as in repeated hypothesis testing: you can trade off accuracy with sample size by changing the interval at which you test. 

![_config.yml]({{ site.baseurl }}/images/2014-11-27/blog_experiments-Copy0_2_1.png)


<!---




















 with 
are the results where I specified the same maximum run-time for all prior results.



    
![_config.yml]({{ site.baseurl }}/images/2014-11-27/Self-Terminating AB Test-Blog_20_1.png)





<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>% lift A over B</th>
      <th>P(Choosing A) Median</th>
      <th>P(Unknown) Median</th>
      <th>Avg Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>-20.0</td>
      <td> 0.000</td>
      <td> 0.000</td>
      <td>   910.75</td>
    </tr>
    <tr>
      <td>-10.0</td>
      <td> 0.005</td>
      <td> 0.000</td>
      <td>  2390.25</td>
    </tr>
    <tr>
      <td> -5.0</td>
      <td> 0.041</td>
      <td> 0.000</td>
      <td>  6098.00</td>
    </tr>
    <tr>
      <td> -2.5</td>
      <td> 0.110</td>
      <td> 0.065</td>
      <td> 12282.50</td>
    </tr>
    <tr>
      <td> -1.5</td>
      <td> 0.159</td>
      <td> 0.137</td>
      <td> 14828.25</td>
    </tr>
    <tr>
      <td>  0.0</td>
      <td> 0.408</td>
      <td> 0.220</td>
      <td> 17409.25</td>
    </tr>
    <tr>
      <td>  1.5</td>
      <td> 0.683</td>
      <td> 0.125</td>
      <td> 14700.00</td>
    </tr>
    <tr>
      <td>  2.5</td>
      <td> 0.814</td>
      <td> 0.072</td>
      <td> 12815.50</td>
    </tr>
    <tr>
      <td>  5.0</td>
      <td> 0.964</td>
      <td> 0.001</td>
      <td>  6305.75</td>
    </tr>
    <tr>
      <td> 10.0</td>
      <td> 0.993</td>
      <td> 0.000</td>
      <td>  2521.50</td>
    </tr>
    <tr>
      <td> 20.0</td>
      <td> 1.000</td>
      <td> 0.000</td>
      <td>  1017.00</td>
    </tr>
  </tbody>
</table>
</div>



The most compelling aspect of this method, however, is that it is very efficient in terms of the number of samples. For example, it is able to find a 5% gain with the same probability as the hypothesis test, but on average, it only requires 40% of the samples. 


    


    


    

-->
    


   
