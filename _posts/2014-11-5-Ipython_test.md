---
layout: post
title: Ipython test
---


 

 
```python
s = "Python syntax highlighting"
print s
```
 
```
No language indicated, so no syntax highlighting. 
But let's throw in a <b>tag</b>.
```


```python
    %matplotlib inline
    %load_ext autoreload
    %autoreload 2
    import pandas as pd
    pd.options.display.mpl_style = 'default'
    
    import inspect, os
    currentdir = os.path.dirname(os.path.abspath(inspect.getfile(inspect.currentframe())))
    parentdir = os.path.dirname(currentdir)
    os.sys.path.insert(0,parentdir) 
    from src.dashboard_helpers import Test, custom_amount_stats
    from src.data_retrieval import *
```


    The autoreload extension is already loaded. To reload it, use:
      %reload_ext autoreload



    t = Test("B14_1027_enUS_ipd_hl_ls", "B14_1027_enUS_ipd_hl_mr", "B14_1027_enUS_mob_hl_ls", "B14_1027_enUS_mob_hl_mr", start = '2014-10-25')


    t.combine(["B14_1027_enUS_ipd_hl_ls", "B14_1027_enUS_mob_hl_ls"], "ls")
    t.combine(["B14_1027_enUS_ipd_hl_mr", "B14_1027_enUS_mob_hl_mr"], "mr")


    t.ecom()




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>donations</th>
      <th>impressions</th>
      <th>dons/i</th>
      <th>amount</th>
      <th>amount/i</th>
      <th>clicks</th>
      <th>clicks/i</th>
      <th>dons/clicks</th>
      <th>amount_ro</th>
      <th>amount_ro/i</th>
      <th>max</th>
      <th>median</th>
      <th>avg</th>
      <th>avg_ro</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>B14_1027_enUS_ipd_hl_ls</th>
      <td> 3447</td>
      <td> 26379700</td>
      <td> 0.000131</td>
      <td> 34909.81</td>
      <td> 0.001323</td>
      <td>  6374</td>
      <td> 0.000242</td>
      <td> 0.540791</td>
      <td> 30907.81</td>
      <td> 0.001172</td>
      <td>  250</td>
      <td> 3</td>
      <td> 10.127592</td>
      <td> 9.069193</td>
    </tr>
    <tr>
      <th>B14_1027_enUS_ipd_hl_mr</th>
      <td> 3543</td>
      <td> 29693800</td>
      <td> 0.000119</td>
      <td> 40996.08</td>
      <td> 0.001381</td>
      <td>  6787</td>
      <td> 0.000229</td>
      <td> 0.522027</td>
      <td> 33896.08</td>
      <td> 0.001142</td>
      <td> 1000</td>
      <td> 3</td>
      <td> 11.571008</td>
      <td> 9.726278</td>
    </tr>
    <tr>
      <th>B14_1027_enUS_mob_hl_ls</th>
      <td> 3860</td>
      <td> 30128000</td>
      <td> 0.000128</td>
      <td> 39638.53</td>
      <td> 0.001316</td>
      <td>  7241</td>
      <td> 0.000240</td>
      <td> 0.533076</td>
      <td> 34579.22</td>
      <td> 0.001148</td>
      <td>  200</td>
      <td> 3</td>
      <td> 10.269049</td>
      <td> 9.073529</td>
    </tr>
    <tr>
      <th>B14_1027_enUS_mob_hl_mr</th>
      <td> 3330</td>
      <td> 26094900</td>
      <td> 0.000128</td>
      <td> 36522.00</td>
      <td> 0.001400</td>
      <td>  6253</td>
      <td> 0.000240</td>
      <td> 0.532544</td>
      <td> 30922.00</td>
      <td> 0.001185</td>
      <td>  300</td>
      <td> 3</td>
      <td> 10.967568</td>
      <td> 9.438950</td>
    </tr>
    <tr>
      <th>ls</th>
      <td> 7307</td>
      <td> 56507700</td>
      <td> 0.000129</td>
      <td> 74548.34</td>
      <td> 0.001319</td>
      <td> 13615</td>
      <td> 0.000241</td>
      <td> 0.536687</td>
      <td> 65539.03</td>
      <td> 0.001160</td>
      <td>  250</td>
      <td> 3</td>
      <td> 10.202318</td>
      <td> 9.077428</td>
    </tr>
    <tr>
      <th>mr</th>
      <td> 6873</td>
      <td> 55788700</td>
      <td> 0.000123</td>
      <td> 77518.08</td>
      <td> 0.001389</td>
      <td> 13040</td>
      <td> 0.000234</td>
      <td> 0.527071</td>
      <td> 64818.08</td>
      <td> 0.001162</td>
      <td> 1000</td>
      <td> 3</td>
      <td> 11.278638</td>
      <td> 9.587055</td>
    </tr>
  </tbody>
</table>
</div>




    t.plot_impressions("ls", "mr",  smooth = 30)


![png](10-27%20enUSlw%20highlight%20combined_files/10-27%20enUSlw%20highlight%20combined_4_0.png)


Looks like ls and mr where served equally. But we know from the individual
analysis, that within the device class, the notebooks were shown at very
different rates


    t.plot_impressions( "B14_1027_enUS_ipd_hl_ls", "B14_1027_enUS_ipd_hl_mr", smooth = 30)


![png](10-27%20enUSlw%20highlight%20combined_files/10-27%20enUSlw%20highlight%20combined_6_0.png)


On the ipad mr was shown more often starting on day 3


    t.plot_impressions( "B14_1027_enUS_mob_hl_ls", "B14_1027_enUS_mob_hl_mr", smooth = 30)


![png](10-27%20enUSlw%20highlight%20combined_files/10-27%20enUSlw%20highlight%20combined_8_0.png)


On mobile, ls was shown more often.

Since we know the distribution over the device class within the combined banner
data ls and mr is not the same, and donation rates depend on the device class,
we cannot do tests on the aggregate data. As an example, the two banners could
be identical, but because one banner was served to more people from a device
class that tends to give more donations, it appears better.


    
