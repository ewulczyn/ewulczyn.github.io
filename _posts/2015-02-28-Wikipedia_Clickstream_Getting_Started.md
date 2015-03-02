---
layout: post
title: Wikipedia Clickstream - Getting Started
---

This post gives an introduction to working with the new February release of the [Wikipedia
Clickstream](http://datahub.io/dataset/wikipedia-clickstream/resource/be85cc68-d1e6-4134-804a-fd36b94dbb82) dataset. The data shows
how people get to a Wikipedia article and what articles they click on next. In
other words,
it gives a weighted network of articles, where each edge weight corresponds to
how often people navigate from one page to another. To give an example, consider
the figure below, which shows incoming and outgoing traffic to the "London"
article.

![_config.yml]({{ site.baseurl }}/images/Wikipedia%20Clickstream%20-%20Getting%20Started_files/Wikipedia%20Clickstream%20-%20Getting%20Started_2_0.png)

The example shows that most people found the "London" page through Google Search
and that only a small fraction of readers went on to another article. Before
diving into some examples of working with the data, let me give a more detailed
explanation of how the data was collected.

###Data Preparation

The data contains counts of _(referer, resource)_ pairs extracted from the
request logs of English Wikipedia. When a client requests a resource by
following a link or performing a search, the URI of the webpage that linked to
the resource is included with the request in an HTTP header called the
"referer". This data captures 22 million _(referer, resource)_ pairs from a
total of 3.2 billion requests collected during the month of February 2015.

The dataset only includes requests for articles in the [main
namespace](https://en.wikipedia.org/wiki/Wikipedia:Namespace) of the desktop
version of English Wikipedia.

Referers were [mapped](https://github.com/ewulczyn/wmf/blob/31279e76525c678e62e2
0f986824120401544b6f/clickstream/oozie/hive_query.sql#L191-L242) to a fixed set
of values corresponding to internal traffic or external traffic from one of the
top 5 global traffic sources to English Wikipedia, based on this scheme:

* an article in the main namespace of English Wikipedia -> the article title
* any Wikipedia page that is not in the main namespace of English Wikipedia -> _other-wikipedia_
* an empty referer -> _other-empty_
* a page from any other Wikimedia project -> _other-internal_
* Google -> _other-google_
* Yahoo -> _other-yahoo_
* Bing -> _other-bing_
* Facebook -> _other-facebook_
* Twitter -> _other-twitter_
* anything else -> _other-other_


MediaWiki Redirects are used to forward clients from one page name to another.
They can be useful if a particular article is referred to by multiple names, or
has alternative punctuation, capitalization or spellings. Requests for pages
that get redirected were mapped to the page they redirect to. For example,
requests for 'Obama' redirect to the 'Barack_Obama' page. Redirects were
resolved using a snapshot of the production [redirects table](https://www.mediawiki.org/wiki/Manual:Redirect_table) from February 28 2015.

Redlinks are links to an article that does not exist. Either the article was
deleted after the creation of the link or the author intended to signal the need
for such an article. Requests for redlinks are included in the data.

We attempt to exclude spider traffic by classifying user agents with the [ua-
parser](https://github.com/tobie/ua-parser) library and a few additonal
Wikipedia specific [filters](https://github.com/ewulczyn/wmf/blob/a541f67c92c93609a028f59d61920fef8a1f425e/clickstream/oozie/hive_query.sql#L47-L51). Furthermore, we attempt to filter out traffic from
bots that request a page and then request all or most of the links on that page
(BFS traversal) by setting a threshold on the rate at which a client can
request articles with the same referer. Requests that were made at too high of
a rate get discarded. For the exact details, see [here](https://github.com/ewulc
zyn/wmf/blob/31279e76525c678e62e20f986824120401544b6f/clickstream/oozie/throttle
_ip_ua_path.py) and [here](https://github.com/ewulczyn/wmf/blob/31279e76525c678e
62e20f986824120401544b6f/clickstream/oozie/throttle_ip_ua_ref.py). The threshold
is quite high to avoid excluding human readers who open tabs as they read. As a
result, requests from slow moving bots are likely to remain in the data. More
sophisticated bot detection that evaluates the clients entire set of requests is
an avenue of future work.

Finally, any _(referer, resource)_ pair with 10 or fewer observations was
removed from the dataset.


### Format
The data includes the following 6 fields:

- **prev_id:** if the referer does not correspond to an article in the main
namespace of English Wikipedia, this value will be empty. Otherwise, it contains
the unique MediaWiki page ID of the article corresponding to the referer i.e.
the previous article the client was on
- **curr_id:** the unique MediaWiki page ID of the article the client requested
- **n:** the number of occurrences of the _(referer, resource)_ pair
- **prev_title:** the result of mapping the referer URL to the fixed set of
values described above
- **curr_title:** the title of the article the client requested
- **type**
    - "link" if the referer and request are both articles and the referer links
to the request
    - "redlink" if the referer is an article and links to the request, but the
request is not in the produiction enwiki.page table
    - "other" if the referer and request are both articles but the referer does
not link to the request. This can happen when clients search or spoof their
refer

#Getting to know the Data

There are various quirks in the data due to the dynamic nature of the network of
articles in English Wikipedia and the prevalence of requests from automata. The
following section gives a brief overview of the data fields and caveats that
need to be kept in mind.



### Loading the Data
First let's load the data into a pandas DataFrame.

~~~python
import pandas as pd
df = pd.read_csv("2015_02_clickstream.tsv", sep='\t', header=0)
#we won't use ids here, so lets discard them
df = df[['prev_title', 'curr_title', 'n', 'type']]
df.columns = ['prev', 'curr', 'n', 'type']
~~~

### Top articles
It has been possible for the public to estimate which pages get the most pageviews per month
from the public [pageview dumps](https://dumps.wikimedia.org/other/pagecounts-raw/) that WMF releases. Unfortunately, there is no
attempt to remove spiders and bots from those dumps. This month the "Layer 2
Tunneling Protocol" was the 3rd most requested article. The logs show that this
article was requested by a small number of clients hundreds of times per minute
within a 4 day window. This kind of request pattern is removed from the
clickstream data, which gives the following as the top 10 pages:

~~~python
df.groupby('curr').sum().sort('n', ascending=False)[:10]
~~~



<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe" align="left">
    <tr>
     
      <!-- <td><img src="http://upload.wikimedia.org/wikipedia/commons/d/de/Wikipedia_Logo_1.0.png" alt="" border=3 height=100 width=100><img></td>-->
      <td> Main_Page</td>
      <td> 127500620</td>
    </tr>
    <tr>
      <!--<td><img src="https://upload.wikimedia.org/wikipedia/en/a/a9/87th_Oscars.jpg" alt="" border=3 height=100 width=100></img></td>-->
      <td>87th_Academy_Awards</td>
      <td>   2559794</td>
    </tr>
    
    <tr>
      <!-- <td><img src="https://upload.wikimedia.org/wikipedia/en/5/5e/50ShadesofGreyCoverArt.jpg" alt="" border=3 height=100 width=100></img></td> -->
      <td>Fifty_Shades_of_Grey</td>
      <td>   2326175</td>
    </tr>
    <tr>
      <td>Alive</td>
      <td>   2244781</td>
    </tr>
    <tr>
      <td>Chris_Kyle</td>
      <td>   1709341</td>
    </tr>
    <tr>
      <td>Fifty_Shades_of_Grey_(film)</td>
      <td>   1683892</td>
    </tr>
    <tr>
      <td>Deaths_in_2015</td>
      <td>   1614577</td>
    </tr>
    <tr>
      <td>Birdman_(film)</td>
      <td>   1545842</td>
    </tr>
    <tr>
      <td>Islamic_State_of_Iraq_and_the_Levant</td>
      <td>   1406530</td>
    </tr>
    <tr>
      <td>Stephen_Hawking</td>
      <td>   1384193</td>
    </tr>
</table>
</div>



The most requested pages tend to be about media that was popular in February. The exceptions are the "Deaths\_in\_2015" article
and the "Alive" disambiguation article. The "Main\_Page" links to "Deaths\_in\_2015" and is the top referer to this article, which would explain the high number of requests. The fact that the "Alive" disambiguation page gets so many hits seems suspsect and is likely to be a fruitfull case to investigate to improve the bot filtering.

### Top Referers
The clickstream data aslo let's us investigate who the top referers to Wikipedia
are:

~~~python
df.groupby('prev').sum().sort('n', ascending=False)[:10]
~~~

<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <tbody>
    <tr>
      <td>other-google</td>
      <td> 1494662520</td>
    </tr>
    <tr>
      <td>other-empty</td>
      <td>  347424627</td>
    </tr>
    <tr>
      <td>other-wikipedia</td>
      <td>  129619543</td>
    </tr>
    <tr>
      <td>other-other</td>
      <td>   77496915</td>
    </tr>
    <tr>
      <td>other-bing</td>
      <td>   65895496</td>
    </tr>
    <tr>
      <td>other-yahoo</td>
      <td>   48445941</td>
    </tr>
    <tr>
      <td>Main_Page</td>
      <td>   29897807</td>
    </tr>
    <tr>
      <td>other-twitter</td>
      <td>   19222486</td>
    </tr>
    <tr>
      <td>other-facebook</td>
      <td>    2312328</td>
    </tr>
    <tr>
      <td>87th_Academy_Awards</td>
      <td>    1680559</td>
    </tr>
  </tbody>
</table>
</div>



The top referer by a large margin is Google. Next comes refererless traffic
(usually clients using HTTPS). Then come other language Wikipedias and pages in
English Wikipedia that are not in the main (i.e. article) namespace. Bing
directs significanlty more traffic to Wikipedia than Yahoo. Social media
referals are tiny compared to Google, with twitter leading 10x more requests
to Wikipedia than Facebook.

### Trending on Social Media
Lets look at what articles were trending on Twitter:

~~~python
df_twitter = df[df['prev'] == 'other-twitter']
df_twitter.groupby('curr').sum().sort('n', ascending=False)[:5]
~~~



<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <tbody>
    <tr>
      <td>Johnny_Knoxville</td>
      <td> 198908</td>
    </tr>
    <tr>
      <td>Peter_Woodcock</td>
      <td> 126259</td>
    </tr>
    <tr>
      <td>2002_Tampa_plane_crash</td>
      <td> 119906</td>
    </tr>
    <tr>
      <td>Sơn_Đoòng_Cave</td>
      <td> 116012</td>
    </tr>
    <tr>
      <td>The_boy_Jones</td>
      <td> 114401</td>
    </tr>
  </tbody>
</table>
</div>



I have no explanations for this, but if you find any of the tweets linking to
these articles, I would be curious to see why they got so many click-throughs.

### Most Requested Missing Pages
Next let's look at the most popular redinks. Redlinks are links to a Wikipedia
page that does not exist, either because it has been deleted, or because the
author is anticipating the creation of the page. Seeing which redlinks are the
most viewed is interesting because it gives some indication about demand for
missing content. Since the set of pages and links is constantly changing, the
labeling of redlinks is not an exact science. In this case, I used a static snapshot of the [page](https://www.mediawiki.org/wiki/Manual:Page_table) and
[pagelinks](https://www.mediawiki.org/wiki/Manual:Pagelinks_table) production tables from Feb 28th to mark a page as a redlink.

~~~python
df_redlinks = df[df['type'] == 'redlink']
df_redlinks.groupby('curr').sum().sort('n', ascending=False)[:5]
~~~


<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <tbody>
    <tr>
      <td>2027_Cricket_World_Cup</td>
      <td> 6782</td>
    </tr>
    <tr>
      <td>Rethinking</td>
      <td> 5279</td>
    </tr>
    <tr>
      <td>Chris_Soules</td>
      <td> 5229</td>
    </tr>
    <tr>
      <td>Anna_Lezhneva</td>
      <td> 3764</td>
    </tr>
    <tr>
      <td>Jillie_Mack</td>
      <td> 3685</td>
    </tr>
  </tbody>
</table>
</div>



### Searching Within Wikipedia

 Usually, clients navigate from one article to another through follwing a link.
The other prominent case is search. The article from which the user searched is
also passed as the referer to the found article. Hence, you will find a high
count of `(Wikipedia, Chris_Kyle)` tuples. People went to the "Wikipedia"
article to search for "Chris\_Kyle". There is not a link to the "Chris\_Kyle"
article from the "Wikipedia" article. Finally, it is possible that the client
messed with their referer header. The vast majority of requests with an internal
referer correspond to a true link.

~~~python
df_search = df[df['type'] == 'other']
df_search =  df_search[df_search.prev.str.match("^other.*").apply(bool) == False]
print "Number of searches/ incorrect referers: %d" % df_search.n.sum()

Number of searches/ incorrect referers: 106772349

df_link = df[df['type'] == 'link']
df_link =  df_link[df_link.prev.str.match("^other.*").apply(bool) == False]
print "Number of links followed: %d" % df_link.n.sum()

Number of links followed: 983436029
~~~

### Inflow vs Outflow

You might be tempted to think that there can't be more traffic coming out of a node (ie. article) than going into a node. This is not true for two reasons. People will
follow links in multiple tabs as they read an article. Hence, a single pageview
can lead to multiple records with that page as the referer. The data is also
certain to include requests from bots which we did not correctly filter out.
Bots will often follow most, if not all, the links in the article. Lets look at
the ratio of incoming to outgoing links for the most requested pages.

~~~python
df_in = df.groupby('curr').sum()  # pageviews per article
df_in.columns = ['in_count',]
df_out = df.groupby('prev').sum() # link clicks per article
df_out.columns = ['out_count',]
df_in_out = df_in.join(df_out)
df_in_out['ratio'] = df_in_out['out_count']/df_in_out['in_count'] #compute ratio if outflow/infow
df_in_out.sort('in_count', ascending = False)[:3]
~~~


<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <tdead>
    <tr style="text-align: left;">
      <td><b>curr</b></td>
      <td><b>in_count</b></td>
      <td><b>out_count</b></td>
      <td><b>ratio</b></td>
    </tr>
  </tdead>
  <tbody>
    <tr>
      <td>Main_Page</td>
      <td> 127500620</td>
      <td> 29897807</td>
      <td> 0.234491</td>
    </tr>
    <tr>
      <td>87th_Academy_Awards</td>
      <td>   2559794</td>
      <td>  1680559</td>
      <td> 0.656521</td>
    </tr>
    <tr>
      <td>Fifty_Shades_of_Grey</td>
      <td>   2326175</td>
      <td>  1146354</td>
      <td> 0.492806</td>
    </tr>
  </tbody>
</table>
</div>



Looking at the pages with the highest ratio of outgoing to incoming traffic
reveals how messy the data is, even after the careful data preparation
described above.

~~~python
df_in_out.sort('ratio', ascending = False)[:3]
~~~



<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: left;">
      <th><b>curr</b></th>
      <th><b>in_count</b></th>
      <th><b>out_count</b></th>
      <th><b>ratio</b></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>List_of_Major_League_Baseball_players_(H)</td>
      <td> 57</td>
      <td> 1323</td>
      <td> 23.210526</td>
    </tr>
    <tr>
      <td>2001–02_Slovak_Superliga</td>
      <td> 22</td>
      <td>  472</td>
      <td> 21.454545</td>
    </tr>
    <tr>
      <td>Principle_of_good_enough</td>
      <td> 23</td>
      <td>  374</td>
      <td> 16.260870</td>
    </tr>
  </tbody>
</table>
</div>



All of these pages have more traversals of a single link than they have requests
for the page to begin with.  As a post processing step, we might enforce that
there can't be more traversals of a link than there were requests to the page.
Better bot filtering should help reduce this issue in the future.

~~~python
df_post = pd.merge(df, df_in, how='left', left_on='prev', right_index=True)
df_post['n'] = df_post[['n', 'in_count']].min(axis=1)
del df_post['in_count']
~~~

###Network Analysis

We can think of Wikipedia as a network with articles as nodes and links between
articles as edges. This network has been available for analysis via the
production pagelinks table. But what does it mean if there is a link between
articles that never gets traversed? What is it about the pages that send their
readers to other pages with high probability? What makes a link enticing to
follow? What are the cliques of articles that send lots of traffic to each
other? These are just some of the questions, this data set allows us to
investigate.  I'm sure you will come up with many more.

### Getting the Data

The dataset is released under [CC0](
https://creativecommons.org/publicdomain/zero/1.0/). The canonical citation and
most up-to-date version of the data can be found at:

Ellery Wulczyn, Dario Taraborelli (2015). Wikipedia Clickstream. *figshare.*
[doi:10.6084/m9.figshare.1305770](http://dx.doi.org/10.6084/m9.figshare.1305770)

An IPython Notebook version of this post can be found [here](http://nbviewer.ipython.org/github/ewulczyn/wmf/blob/dd241618811536975b1837a654270b9614b7af2a/clickstream/ipython/Wikipedia%20Clickstream%20-%20Getting%20Started.ipynb). Note that you will have to download the notebook and re-execute the code cells to get the output. 

