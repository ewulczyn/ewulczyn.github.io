---
layout: page
title: Projects
permalink: /projects/
redirect_from:
  - /index.html
---


### DeTox 
A collection of machine learning models for detecting personal attacks and aggression in discussion comments. We are using the models to study the prevalence and impact of toxic comments on Wikipedia, as well as encourage the development of new moderation tools. In collaboration with [Google Jigsaw](https://jigsaw.google.com). This is work in progress.

[[homepage](https://meta.wikimedia.org/wiki/Research:Detox/Research)] [[code](https://github.com/ewulczyn/wiki-detox)] [[app](https://wikidetox.appspot.com/)]

___

### Wikipedia Navigation Vectors

An embedding of Wikipedia articles and Wikidata items by applying Word2vec models to a corpus of billions of reading sessions. The embeddings have the property that articles that tend to be read in close succession have similar vector representations. A demo app of how these vectors can be used to generate reading recommendations is linked below.

[[homepage](https://meta.wikimedia.org/wiki/Research:Wikipedia_Navigation_Vectors)] [[code](https://github.com/ewulczyn/wiki-vectors)] [[data](https://meta.wikimedia.org/wiki/Research:Wikipedia_Navigation_Vectors)] [[app](https://tools.wmflabs.org/readmore/)]

___

### GapFinder 

System for recommending articles for translation between Wikipedias. Nominated for best paper at WWW 2016.

[[paper] (http://arxiv.org/abs/1604.03235)] [[video](https://www.youtube.com/watch?v=xnn5_ObBk2o)] [[code](https://github.com/ewulczyn/wiki-gapfinder)] [[app](http://recommend.wmflabs.org)]

___


### Wikipedia Clickstream

Dataset containing counts of billions of (referer, resource) pairs extracted from the request logs of Wikipedia. The data shows how people get to a Wikipedia article and what links they click on. In other words, it gives a weighted network of articles, where each edge weight corresponds to how often people navigate from one page to another.

[[homepage](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream)] [[code](https://github.com/ewulczyn/wiki-clickstream)] [[data](https://figshare.com/articles/Wikipedia_Clickstream/1305770)]

___

### Practical AB testing

A collection of blog posts containing practical advice on AB testing and some useful extensions to Bayesian hypothesis testing methods based on learnings from hundreds of tests at WMF.


- <a href="{{ site.baseurl }}/ab_testing_and_independence">AB Testing and the Importance of Independent Observations</a>

- <a href="{{ site.baseurl }}/what_if_ab_testing_is_like_science">What if AB Testing is like Science ... and most results are false?</a>

- <a href="{{ site.baseurl }}/ab_testing_with_multinomial_data">Beautifully Bayesian - AB Testing with Discrete Rewards</a>

- <a href="{{ site.baseurl }}/how_naive_ab_testing_goes_wrong/">How Naive AB Testing Goes Wrong and How to Fix It</a>


___

### Mining for Earmarks

Built a machine learning system for automatically extracting earmarks from congressional bills and reports. The system was used to construct the first publicly available database of earmarks dating back to 1995. Won the runner-up prize at the 2015 Bloomberg Data for Good Exchange. Accepted at KDD 2016.

[[homepage](https://dssg.uchicago.edu/earmarks)] [<a href="{{ site.baseurl }}/images/identifying-earmarks-congressional.pdf">paper</a>] [[code](https://github.com/dssg/machine_learning_legislation)] [[data](https://dssg.uchicago.edu/earmarks)]

___

### Deep Learning for Text Classification

Recursive Neural Network for Short Text Classification. The model jointly learns vector representations of words, a method of merging word vectors into document vectors and a classifier over the document vectors. Implemented from scratch back in the day before we had deep learning frameworks.

[[paper](http://clementinejacoby.com/softmax_rnn_224.pdf)]

___

### Trust in the CouchSurfing Network

Inference of trust ratings between strangers from trust ratings
between acquaintances and the structure of the network
that connects them. My CS229 project turned ICWSM paper.

[[paper](https://web.stanford.edu/~cgpotts/papers/OvergoorWulczynPotts.pdf)]
