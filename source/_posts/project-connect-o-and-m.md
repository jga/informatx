---
title: Project Connect O&M
date: 2019-11-13 08:00:00
tags: ["transit"]
---

On October 30th, Project Connect provided a joint meeting of the Austin City Council and its Board of Directors two memos estimating the operations and maintenance (O&M) costs for Bus Rapid Transit (BRT) and Light Rail Transit (LRT) modes serving the alignments commonly referred to as the "Orange" and "Blue" Lines.

This analysis examines the *revenue hour* cost estimates in those memos, as they are the foundation of the overall O&M cost estimate.

<!-- more -->

There are two main findings:

- Project Connect's BRT revenue hour cost estimate is lower than the national average by 26%. Project Connect does not explain its rationale for the methodological choices that lead to the lower rate.

- Project Connect's use of a *flat* passenger car revenue hour rate to calculate LRT costs obfuscates the economies of scale associated with multi-car LRT trains. This is a change from the approach taken by Project Connect in 2013-2014. The new method makes Blue Line LRT appear more productive and Orange Line LRT less productive than an approach that recognizes the cost advantages of LRT scale (e.g. multi-car trains). Project Connect does not explain the rationale for the methodological switch or why its current approach will generate more accurate estimates.

The conclusion to this analysis provides several recommendations for Project Connect personnel, as well as local media, and activists.

To provide transparency and clarity about the data and methods used, this analysis is presented as a [Jupyter Notebook](https://jupyter.org/). This format allows for Project Connect vendors, local media, and activists to easily edit, correct, or extend the analysis.

## Setting up our Notebook

Before we get to doing any actual work with data, we have to import the packaged libraries of code that we will use to conduct our analysis. Using already-existing libraries dramatically cuts down on the amount of code one has to write.

The Python packages we are using are popular and well-documented; they will allows us to efficiently read, organize, and manipulate CSV data, as well as run statistical functions and visualize results.


```python
import pandas as pd
import numpy as np
from numpy.polynomial.polynomial import polyfit
from scipy.stats import variation
import matplotlib.pyplot as plt
import statsmodels.api as sm
```

## BRT O&M

The main finding of this review of Project Connect's BRT O&M calculations is that its BRT revenue hour figure is **lower** when compared to the average of the BRT implementations reported in the National Transit Database (NTD).

### Project Connect's BRT Revenue Hour

On page 15 of the October 30th Orange Line O&M memo, Project Connect provides this explanation of their cost estimation methodology:

{% blockquote %}

    Unit costs are given in 2028 dollars, reflecting the anticipated opening date for the Orange Line, and are escalated 3% annually for estimates associated with the 2040 ridership forecast year.

{% endblockquote %}

The language on the Orange Line memo leaves the question of whether the batch of 2028 estimates within it are also inflated at the annual 3%. The Blue Line O&M memo is, thankfully, worded in a clearer fashion. Its page 13 explanation removes the doubt:

{% blockquote %}

    Cost calculations are mode-specific and presented in 2028 dollars reflecting the anticipated opening year for the Blue Line Corridor. Unit costs were inflated at three percent annually to 2040.

{% endblockquote %}

In addition to the 3% annual cost inflation constant, the Orange and Blue Line memos provide the following details about the BRT revenue hour figure:

{% blockquote %}

    For new bus and BRT services not covered by the contract, a unit cost of $156.93 per revenue hour (2028) is used as a fully allocated cost (including fuel and general administration). For BRT, an additional cost factor for guideway maintenance is added to the cost per revenue hour. Based on National Transit Database information, an adjustment of $30,000 per directional guideway mile was applied to street-level guideway options. For mixed and elevated guideway options, an annual cost of $80,000 per directional guideway mile was used.

{% endblockquote %}

The language implies that the Project Connect team is using a current, contractually-negotiated set of numbers for the three expense categories the 2018 and 2019 NTD Full Reporting Policy Manuals label as "vehicle operations", "vehicle maintenance", and "general administration". And then Project Connect switches to using NTD products to figure out the guideway maintenance, which is typically included under the NTD operating expense reporting category labeled as "non-vehicle maintenance".

Unfortunately, it is not clear what method was used to calculate the $156.93 number.

The Orange Line memo includes a set of tables under Appendix A on page 16. The 2028 O&M Cost Calculations in that table report the vehicle revenue hours and total annual operating expenses under two guideway options. The figures are 173,000 revenue hours and $30,249,000 cost for the elevated option. It's 148,000 revenue hours and $24,399,000 cost for the surface option.

Let's calculate the actual 2028 revenue hour cost including the guideway maintenance.



```python
brt_revenue_hour_28_elevated = 30_249_000 / 173_000
brt_revenue_hour_28_surface = 24_399_000 / 148_000
print('BRT Elevated Option Revenue Hour (2028): ${:.2f}'.format(brt_revenue_hour_28_elevated))
print('BRT Surface Option Revenue Hour (2028): ${:.2f}'.format(brt_revenue_hour_28_surface))
```

<pre>
    BRT Elevated Option Revenue Hour (2028): $174.85
    BRT Surface Option Revenue Hour (2028): $164.86
</pre>

Now, let's change those 2028 dollars to 2017 dollars (the NTD data year used by Project Connect) using the 3% annual rate identified in their methodology narrative.


```python
brt_revenue_hour_17_elevated = brt_revenue_hour_28_elevated / (1.03 ** 11)
brt_revenue_hour_17_surface = brt_revenue_hour_28_surface / (1.03 ** 11)
print('BRT Elevated Option Revenue Hour (2017): ${:.2f}'.format(brt_revenue_hour_17_elevated))
print('BRT Surface Option Revenue Hour (2017): ${:.2f}'.format(brt_revenue_hour_17_surface))
```
<pre>
    BRT Elevated Option Revenue Hour (2017): $126.32
    BRT Surface Option Revenue Hour (2017): $119.10
</pre>

### NTD's BRT Revenue Hour

Are Project Connect's \$126 and \$119 revenue hour costs for 2017 correct?

To double-check whether those numbers make sense, we will utilize the available 2017 BRT data from the NTD.

Specifically, we transform the 2017 operating expense and service data Excel spreadsheets accessible by the [NTD Data Reports download page](https://www.transit.dot.gov/ntd/ntd-data) into easier to consume CSV files accessible through this site.

To begin our BRT revenue hour review, we load two CSVs of 2017 NTD operating data.

The first CSV file includes the vehicle revenue hours for each agency and mode for 2017. The second CSV file includes the *total* operating expenses for each agency and mode. As discussed above, the *total* operating expense amounts include non-vehicle maintenance, such as guideway maintenance.

You'll start to see the letters "df" quite a bit at this point in the Notebook. They are used to help the reader understand that the variable is a DataFrame. For those of you unfamiliar with dataframes/Python data science tools, think of a dataframe as something similar to a spreadsheet or an Excel Worksheet. A dataframe, like an Excel Worksheet, is used to represent and efficiently work with rows and columns of data.

Once the data files are loaded, we make a query to select only the data for operating Bus Rapid Transit modes. NTD uses the initials `RB` to label the BRT mode.

We do some column name cleanup to facilitate legibility and then we merge the vehicle revenue hours (VRH) and operating expenses (OPEX) data sets. This allows us to create a new column providing the 2017 Revenue Hour Cost.


```python
vehicle_revenue_hour_df = pd.read_csv('ntd_vrh_2017.csv')
opex_df = pd.read_csv('ntd_total_opex_2017.csv')
brt_vehicle_revenue_hour_df = vehicle_revenue_hour_df.query("Mode=='RB' & `Mode Status` == 'Operating'")
brt_vehicle_revenue_hour_df = brt_vehicle_revenue_hour_df.rename(columns={'2017': '2017 VRH'})
brt_opex_df = opex_df.query("Mode=='RB' & `Mode Status` == 'Operating'")
brt_opex_df = brt_opex_df.rename(columns={'2017': '2017 OPEX'})
bare_brt_opex_df = brt_opex_df[['NTD ID', 'Service', '2017 OPEX']].copy()
brt_df = pd.merge(brt_vehicle_revenue_hour_df, bare_brt_opex_df, on=['NTD ID', 'Service'])
brt_df['Revenue Hour Cost'] = brt_df['2017 OPEX'] / brt_df['2017 VRH']
brt_df[['Agency Name', 'City', '2017 OPEX', '2017 VRH', 'Revenue Hour Cost']]
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th>Agency Name</th>
      <th>City</th>
      <th>2017 OPEX</th>
      <th>2017 VRH</th>
      <th>Revenue Hour Cost</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Lane Transit District(LTD)</td>
      <td>Eugene</td>
      <td>6463202.0</td>
      <td>37929.0</td>
      <td>170.402647</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Massachusetts Bay Transportation Authority(MBTA)</td>
      <td>Boston</td>
      <td>18489929.0</td>
      <td>125579.0</td>
      <td>147.237428</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Connecticut Department of Transportation - CTT...</td>
      <td>Hartford</td>
      <td>9105566.0</td>
      <td>41851.0</td>
      <td>217.571050</td>
    </tr>
    <tr>
      <th>3</th>
      <td>MTA New York City Transit(NYCT)</td>
      <td>New York</td>
      <td>99445139.0</td>
      <td>527568.0</td>
      <td>188.497291</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Central Florida Regional Transportation Author...</td>
      <td>Orlando</td>
      <td>3290958.0</td>
      <td>46974.0</td>
      <td>70.059139</td>
    </tr>
    <tr>
      <th>5</th>
      <td>The Greater Cleveland Regional Transit Authori...</td>
      <td>Cleveland</td>
      <td>6161689.0</td>
      <td>67204.0</td>
      <td>91.686343</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Interurban Transit Partnership(The Rapid)</td>
      <td>Grand Rapids</td>
      <td>2098600.0</td>
      <td>27793.0</td>
      <td>75.508221</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Kansas City Area Transportation Authority(KCATA)</td>
      <td>Kansas City</td>
      <td>5637075.0</td>
      <td>44835.0</td>
      <td>125.729341</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Transfort</td>
      <td>Fort Collins</td>
      <td>3115771.0</td>
      <td>30069.0</td>
      <td>103.620706</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Roaring Fork Transportation Authority</td>
      <td>Westcliffe</td>
      <td>8506328.0</td>
      <td>69681.0</td>
      <td>122.075286</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Los Angeles County Metropolitan Transportation...</td>
      <td>Los Angeles</td>
      <td>32258690.0</td>
      <td>123183.0</td>
      <td>261.876152</td>
    </tr>
  </tbody>
</table>
</div>



As of the 2017 NTD, there are 11 agencies running BRT.

Interestingly, Project Connect predicts just the Orange Line's required vehicle revenue hours in 2028 will already exceed the 2017 BRT vehicle revenue hour amounts of each *agency* except New York's MTA.

Here's what we get as the national average of all revenue hours and costs across the 11 agencies for 2017.


```python
national_brt_revenue_hour_avg = brt_df['2017 OPEX'].sum() / brt_df['2017 VRH'].sum()
print('National BRT revenue hour average (2017): ${:.2f}'.format(national_brt_revenue_hour_avg))
```

<pre>
    National BRT revenue hour average (2017): $170.28
</pre>

The \$170 (2017) national average is higher than Project Connect's upper estimate of \$126 (2017). Project Connect's revenue hour estimate is 26% lower than the national average.

As indicated above, while the Project Connect team detailed *how* they created their revenue hour estimate, it didn't explain *why* it made the choices it did relative to just selecting the national average.

One argument against using the national average is that there are no natural peers in the NTD BRT data. There are no Sun Belt or non-coastal agencies that are running the scale of BRT revenue hours envisioned by Project Connect for the Orange and Blue Lines. On the other hand, the systems running a lot of BRT hours have the higher per hour costs.

There might be unforeseen vehicle and non-vehicle maintenance costs when true BRT is used at levels not presently experienced by CapMetro. Simply asserting a revenue hour estimate without establishing its validity undermines quality decision-making by policymakers.

## LRT O&M

The main finding of this review of Project Connect's LRT O&M calculations is that it obscures the scaling advantages of LRT.

This undermines the productivity of Orange Line LRT, while inflating that of Blue Line LRT.

### Project Connect's LRT Revenue Hour

On page 15 of the October 30th Orange Line O&M memo, Project Connect provides the following explanation for their LRT revenue hour cost:

{% blockquote %}

    O&M unit costs for LRT service reflect a weighted national average cost per revenue hour of $393.33 (2028, adjusted from $284.15 2017 NTD). This includes guideway maintenance.

{% endblockquote %}

The Blue Line LRT estimates use the same method. The 3% annual inflation constant is also used.

One important decision that Project Connect's new methodology makes is that it views each passenger car in a train as a "vehicle" with a fixed cost.

So, the third car on the train costs as much as the first one.

During the 2013-14 version of Project Connect, its vendors used a [different methodology](http://keepaustinwonky.org) that recognized fixed system costs, train costs, and passenger car costs.

We'll discuss this choice and its implications in more detail below.

### NTD's LRT Revenue Hour

Is the $284.15 revenue hour cost for 2017 correct?

To check Project Connect's number, we will use the available 2017 LRT data from the NTD.

Again, we transform the 2017 operating expense and service data Excel spreadsheets accessible by the [NTD Data Reports download page](http://www.transit.dot.gov/ntd/ntd-data) into easier to consume CSV files accessible through this site.

We already loaded the necessary data to conduct the above BRT analysis, so we query to select only the data for operating LRT modes, and then do the same data cleanup and merging as we did with the BRT section above.

```python
lrt_vehicle_revenue_hour_df = vehicle_revenue_hour_df.query("Mode=='LR' & `Mode Status` == 'Operating'")
lrt_vehicle_revenue_hour_df = lrt_vehicle_revenue_hour_df.rename(columns={'2017': '2017 VRH'})
lrt_opex_df = opex_df.query("Mode=='LR' & `Mode Status` == 'Operating'")
lrt_opex_df = lrt_opex_df.rename(columns={'2017': '2017 OPEX'})
bare_lrt_opex_df = lrt_opex_df[['NTD ID', 'Service', '2017 OPEX']].copy()
lrt_df = pd.merge(lrt_vehicle_revenue_hour_df, bare_lrt_opex_df, on=['NTD ID', 'Service'])
lrt_df['Revenue Hour Cost'] = lrt_df['2017 OPEX'] / lrt_df['2017 VRH']
lrt_df[['Agency Name', 'City', 'Service', '2017 OPEX', '2017 VRH', 'Revenue Hour Cost']]
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th>Agency Name</th>
      <th>City</th>
      <th>Service</th>
      <th>2017 OPEX</th>
      <th>2017 VRH</th>
      <th>Revenue Hour Cost</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Tri-County Metropolitan Transportation Distric...</td>
      <td>Portland</td>
      <td>DO</td>
      <td>138797386.0</td>
      <td>623791.0</td>
      <td>222.506234</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Central Puget Sound Regional Transit Authority...</td>
      <td>Seattle</td>
      <td>DO</td>
      <td>91194100.0</td>
      <td>251376.0</td>
      <td>362.779661</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Massachusetts Bay Transportation Authority(MBTA)</td>
      <td>Boston</td>
      <td>DO</td>
      <td>187119893.0</td>
      <td>668402.0</td>
      <td>279.951127</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Niagara Frontier Transportation Authority(NFT ...</td>
      <td>Buffalo</td>
      <td>DO</td>
      <td>23862248.0</td>
      <td>82845.0</td>
      <td>288.034860</td>
    </tr>
    <tr>
      <th>4</th>
      <td>New Jersey Transit Corporation(NJ TRANSIT)</td>
      <td>Newark</td>
      <td>DO</td>
      <td>19726085.0</td>
      <td>51556.0</td>
      <td>382.614730</td>
    </tr>
    <tr>
      <th>5</th>
      <td>New Jersey Transit Corporation(NJ TRANSIT)</td>
      <td>Newark</td>
      <td>PT</td>
      <td>92685527.0</td>
      <td>123198.0</td>
      <td>752.329802</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Port Authority of Allegheny County(Port Author...</td>
      <td>Pittsburgh</td>
      <td>DO</td>
      <td>62950866.0</td>
      <td>169646.0</td>
      <td>371.071914</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Maryland Transit Administration(MTA)</td>
      <td>Baltimore</td>
      <td>DO</td>
      <td>42626110.0</td>
      <td>157002.0</td>
      <td>271.500427</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Transportation District Commission of Hampton ...</td>
      <td>Hampton</td>
      <td>DO</td>
      <td>11609880.0</td>
      <td>29868.0</td>
      <td>388.706308</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Charlotte Area Transit System(CATS)</td>
      <td>Charlotte</td>
      <td>DO</td>
      <td>14291472.0</td>
      <td>66641.0</td>
      <td>214.454645</td>
    </tr>
    <tr>
      <th>10</th>
      <td>The Greater Cleveland Regional Transit Authori...</td>
      <td>Cleveland</td>
      <td>DO</td>
      <td>12741916.0</td>
      <td>50655.0</td>
      <td>251.543105</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Metro Transit</td>
      <td>Minneapolis</td>
      <td>DO</td>
      <td>70924094.0</td>
      <td>426665.0</td>
      <td>166.228995</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Metropolitan Transit Authority of Harris Count...</td>
      <td>Houston</td>
      <td>DO</td>
      <td>65168737.0</td>
      <td>287042.0</td>
      <td>227.035545</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Dallas Area Rapid Transit(DART)</td>
      <td>Dallas</td>
      <td>DO</td>
      <td>175197867.0</td>
      <td>491854.0</td>
      <td>356.198927</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Bi-State Development Agency of the Missouri-Il...</td>
      <td>St. Louis</td>
      <td>DO</td>
      <td>76332879.0</td>
      <td>264889.0</td>
      <td>288.169305</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Utah Transit Authority(UTA)</td>
      <td>Salt Lake City</td>
      <td>DO</td>
      <td>64680283.0</td>
      <td>358645.0</td>
      <td>180.346256</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Denver Regional Transportation District(RTD)</td>
      <td>Denver</td>
      <td>DO</td>
      <td>115181118.0</td>
      <td>793856.0</td>
      <td>145.090694</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Santa Clara Valley Transportation Authority(VTA)</td>
      <td>San Jose</td>
      <td>DO</td>
      <td>106017010.0</td>
      <td>217434.0</td>
      <td>487.582485</td>
    </tr>
    <tr>
      <th>18</th>
      <td>San Francisco Municipal Railway(MUNI)</td>
      <td>San Francisco</td>
      <td>DO</td>
      <td>213773526.0</td>
      <td>579417.0</td>
      <td>368.945899</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Sacramento Regional Transit District(Sacrament...</td>
      <td>Sacramento</td>
      <td>DO</td>
      <td>67841139.0</td>
      <td>248913.0</td>
      <td>272.549602</td>
    </tr>
    <tr>
      <th>20</th>
      <td>San Diego Metropolitan Transit System(MTS)</td>
      <td>San Diego</td>
      <td>DO</td>
      <td>82472931.0</td>
      <td>490197.0</td>
      <td>168.244463</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Los Angeles County Metropolitan Transportation...</td>
      <td>Los Angeles</td>
      <td>DO</td>
      <td>366354851.0</td>
      <td>789513.0</td>
      <td>464.026369</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Valley Metro Rail, Inc.(VMR)</td>
      <td>Phoenix</td>
      <td>PT</td>
      <td>41487087.0</td>
      <td>265331.0</td>
      <td>156.359743</td>
    </tr>
  </tbody>
</table>
</div>




```python
national_lrt_revenue_hour_avg = lrt_df['2017 OPEX'].sum() / lrt_df['2017 VRH'].sum()
print('National LRT revenue hour average (2017): ${:.2f}'.format(national_lrt_revenue_hour_avg))
```
<pre>
    National LRT revenue hour average (2017): $286.17
</pre>

The calculated national LRT revenue hour average of \$286.17 is virtually identical to Project Connect's \$284.15.

Unlike BRT, though, there is a set of communities that could arguably be seen as more closely resembling central Texas' labor policy and business costs, and therefore serve as a more accurate proxy for this region's likely O&M costs. During the 2013-2014 version of Project Connect, their methodology used a peer subset to calculate LRT O&M costs.

Specifically, Houston, Denver, Salt Lake City, Dallas, Charlotte, and Phoenix are all non-coastal communities with labor, state policy and business cost drivers that are perceived by many as more similar to Austin's that the main West Coast metros or Northeast cities.

Let's see what the peer city LRT revenue hour looks like.


```python
peer_cities = {'Houston', 'Denver', 'Salt Lake City', 'Dallas', 'Charlotte', 'Phoenix'}
peer_lrt_df = lrt_df[lrt_df['City'].isin(peer_cities)]
peer_lrt_revenue_hour_avg = peer_lrt_df['2017 OPEX'].sum() / peer_lrt_df['2017 VRH'].sum()
national_lrt_variation = variation(lrt_df['Revenue Hour Cost'])
peer_lrt_variation = variation(peer_lrt_df['Revenue Hour Cost'])
print('Peer city LRT revenue hour average (2017): ${:.2f}'.format(peer_lrt_revenue_hour_avg))
print('Peer city LRT revenue hour discount factor (2017): {}'.format(peer_lrt_revenue_hour_avg / national_lrt_revenue_hour_avg))
print('National LRT revenue hour coefficient of variation: {}'.format(national_lrt_variation))
print('Peer city LRT revenue hour coefficient of variation: {}'.format(peer_lrt_variation))


```

<pre>
    Peer city LRT revenue hour average (2017): $210.31
    Peer city LRT revenue hour discount factor (2017): 0.7349138441935215
    National LRT revenue hour coefficient of variation: 0.43697372476675667
    Peer city LRT revenue hour coefficient of variation: 0.3293199499498427
</pre>

As expected, the peer city LRT is lower at \$210.31 (2017), with the peer city average revenue hour representing 76% of the national average revenue hour cost.

The peer city revenue hour cost estimates have a lower [coefficient of variation](https://en.wikipedia.org/wiki/Coefficient_of_variation) (a common measure of dispersion) than the entire national LRT data set.

Why does this matter? A reasonable argument against using a smaller data set of peer cities to select the revenue hour cost figure would be to raise concern about how dramatic outliers could easily distort the average. That the peer city data has a smaller coefficient of variation than the complete LRT data set addresses the outlier concern.

### Train vs. Car

NTD's Full Reporting Policy Manuals and its actual database make a distinction between a light rail *train* revenue hour and a passenger *car* revenue hour. As explained above, a train can have one or more cars.

Why would NTD collect car *and* train data if a flat passenger car revenue hour is the superior budgeting metric, as Project Connect assumes in its October 30th O&M memos? Why would they track train hours?

There are several reasons, but one of them is the perceived scaling advantage of light rail. From an O&M perspective, a common assumption shared by some industry professionals and transit advocates is that the fixed expenses associated with fielding a train with one car (e.g. the train operator) can be shared by additional cars. The implication is that the cost of serving an additional rail customer with light rail keeps going down as passenger cars are added.

This is different than BRT, which must field an additional vehicle with, for example, an additional driver. With BRT, the "train" always has one "car".

So, are the perceived advantages of rail at scale real or is it just theory?

Let's dive into this question by loading a new data set that includes *train* revenue hours. In this data set, an individual agency can have data associated with the light rail services it directly operates ("DO" service type) or that it pays for and does not directly operate ("PT" service type).

We'll do some cleanup on the column names and drop the Hudson-Bergen light rail as it is an outlier data point with a much, much higher cost structure and it is not directly operated by a transit agency. And we'll merge the new train data with the already loaded agency operating expense data. We also mark the peer cities discussed above with a [dummy variable](https://en.wikipedia.org/wiki/Dummy_variable_(statistics)).

After some basic calculations, there are four new columns for each agency's service implementation.

*Extra Car Revenue Hours* finds the difference between train revenue hours and vehicle revenue hours, which indicates the count of revenue hours produced by the cars operated in addition to the first train car.

The *Load Factor* divides the total vehicle revenue hours by the train revenue hours; this helps us determine the average number of cars for a train.

*Train RevHr Opex* divides the total 2017 operating expenses by *train* revenue hours. This provides an estimate of what the average LRT train costs to run for each agency service type.

*Car RevHr Opex* divides the total 2017 operating expenses by vehicle revenue hours. A "vehicle" is a passenger car according to the NTD reporting manual. This column reports the average LRT car costs to run for each agency service type.

We sort the data table by load factor in ascending order.


```python
service_df = pd.read_csv('ntd_service_2017.csv')
rail_df = service_df.query("Mode=='LR'").copy()
rail_df = rail_df.rename(columns={'Train \nRevenue \nHours': 'Train Revenue Hrs',
                                  'Vehicle \nRevenue \nHours': 'Vehicle Revenue Hrs',
                                  'Type of Service': 'Service'})
rail_df = rail_df.drop(rail_df[(rail_df['Service'] == 'PT') & (rail_df['City'] == 'Newark')].index)
rail_df['Extra Car Revenue Hrs'] = rail_df['Vehicle Revenue Hrs'] - rail_df['Train Revenue Hrs']
rail_df['Load Factor'] = rail_df['Vehicle Revenue Hrs'] / rail_df['Train Revenue Hrs']
rail_df['NTD ID'] = rail_df['NTD ID'].astype(int)
rail_economics_df = pd.merge(rail_df, bare_lrt_opex_df, on=['NTD ID', 'Service'])
peer_cities = {'Houston', 'Denver', 'Salt Lake City', 'Dallas', 'Charlotte', 'Phoenix'}
rail_economics_df['Peer'] = np.where(rail_economics_df['City'].isin(peer_cities), 1, 0)
rail_economics_df['Train RevHr Opex'] = rail_economics_df['2017 OPEX'] / rail_economics_df['Train Revenue Hrs']
rail_economics_df['Car RevHr Opex'] = rail_economics_df['2017 OPEX'] / rail_economics_df['Vehicle Revenue Hrs']
rail_economics_df[['Name', 'City', 'Peer', 'Service', 'Train Revenue Hrs', 'Vehicle Revenue Hrs', 'Extra Car Revenue Hrs', 'Train RevHr Opex', 'Car RevHr Opex', 'Load Factor']].sort_values(by=['Load Factor'])
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th>Name</th>
      <th>City</th>
      <th>Peer</th>
      <th>Service</th>
      <th>Train Revenue Hrs</th>
      <th>Vehicle Revenue Hrs</th>
      <th>Extra Car Revenue Hrs</th>
      <th>Train RevHr Opex</th>
      <th>Car RevHr Opex</th>
      <th>Load Factor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>New Jersey Transit Corporation</td>
      <td>Newark</td>
      <td>0</td>
      <td>DO</td>
      <td>51556</td>
      <td>51556</td>
      <td>0</td>
      <td>382.614730</td>
      <td>382.614730</td>
      <td>1.000000</td>
    </tr>
    <tr>
      <th>18</th>
      <td>Transportation District Commission of Hampton ...</td>
      <td>Hampton</td>
      <td>0</td>
      <td>DO</td>
      <td>29868</td>
      <td>29868</td>
      <td>0</td>
      <td>388.706308</td>
      <td>388.706308</td>
      <td>1.000000</td>
    </tr>
    <tr>
      <th>14</th>
      <td>The Greater Cleveland Regional Transit Authority</td>
      <td>Cleveland</td>
      <td>0</td>
      <td>DO</td>
      <td>50272</td>
      <td>50655</td>
      <td>383</td>
      <td>253.459500</td>
      <td>251.543105</td>
      <td>1.007619</td>
    </tr>
    <tr>
      <th>8</th>
      <td>San Francisco Municipal Railway</td>
      <td>San Francisco</td>
      <td>0</td>
      <td>DO</td>
      <td>395030</td>
      <td>579417</td>
      <td>184387</td>
      <td>541.157699</td>
      <td>368.945899</td>
      <td>1.466767</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Metropolitan Transit Authority of Harris Count...</td>
      <td>Houston</td>
      <td>1</td>
      <td>DO</td>
      <td>194682</td>
      <td>287042</td>
      <td>92360</td>
      <td>334.744542</td>
      <td>227.035545</td>
      <td>1.474415</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Port Authority of Allegheny County</td>
      <td>Pittsburgh</td>
      <td>0</td>
      <td>DO</td>
      <td>114991</td>
      <td>169646</td>
      <td>54655</td>
      <td>547.441678</td>
      <td>371.071914</td>
      <td>1.475298</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Santa Clara Valley Transportation Authority</td>
      <td>San Jose</td>
      <td>0</td>
      <td>DO</td>
      <td>139433</td>
      <td>217434</td>
      <td>78001</td>
      <td>760.343749</td>
      <td>487.582485</td>
      <td>1.559416</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Massachusetts Bay Transportation Authority</td>
      <td>Boston</td>
      <td>0</td>
      <td>DO</td>
      <td>388003</td>
      <td>668402</td>
      <td>280399</td>
      <td>482.264037</td>
      <td>279.951127</td>
      <td>1.722672</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Maryland Transit Administration</td>
      <td>Baltimore</td>
      <td>0</td>
      <td>DO</td>
      <td>87951</td>
      <td>157002</td>
      <td>69051</td>
      <td>484.657480</td>
      <td>271.500427</td>
      <td>1.785108</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Dallas Area Rapid Transit</td>
      <td>Dallas</td>
      <td>1</td>
      <td>DO</td>
      <td>268769</td>
      <td>491854</td>
      <td>223085</td>
      <td>651.852956</td>
      <td>356.198927</td>
      <td>1.830025</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Tri-County Metropolitan Transportation Distric...</td>
      <td>Portland</td>
      <td>0</td>
      <td>DO</td>
      <td>312847</td>
      <td>623791</td>
      <td>310944</td>
      <td>443.658996</td>
      <td>222.506234</td>
      <td>1.993917</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Charlotte Area Transit System</td>
      <td>Charlotte</td>
      <td>1</td>
      <td>DO</td>
      <td>33412</td>
      <td>66641</td>
      <td>33229</td>
      <td>427.734706</td>
      <td>214.454645</td>
      <td>1.994523</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Bi-State Development Agency of the Missouri-Il...</td>
      <td>St. Louis</td>
      <td>0</td>
      <td>DO</td>
      <td>132444</td>
      <td>264889</td>
      <td>132445</td>
      <td>576.340786</td>
      <td>288.169305</td>
      <td>2.000008</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Valley Metro Rail, Inc.</td>
      <td>Phoenix</td>
      <td>1</td>
      <td>PT</td>
      <td>125474</td>
      <td>265331</td>
      <td>139857</td>
      <td>330.642898</td>
      <td>156.359743</td>
      <td>2.114629</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Utah Transit Authority</td>
      <td>Salt Lake City</td>
      <td>1</td>
      <td>DO</td>
      <td>161136</td>
      <td>358645</td>
      <td>197509</td>
      <td>401.401816</td>
      <td>180.346256</td>
      <td>2.225729</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Los Angeles County Metropolitan Transportation...</td>
      <td>Los Angeles</td>
      <td>0</td>
      <td>DO</td>
      <td>334981</td>
      <td>789513</td>
      <td>454532</td>
      <td>1093.658599</td>
      <td>464.026369</td>
      <td>2.356889</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Central Puget Sound Regional Transit Authority</td>
      <td>Seattle</td>
      <td>0</td>
      <td>DO</td>
      <td>96191</td>
      <td>251376</td>
      <td>155185</td>
      <td>948.052313</td>
      <td>362.779661</td>
      <td>2.613301</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Sacramento Regional Transit District</td>
      <td>Sacramento</td>
      <td>0</td>
      <td>DO</td>
      <td>94858</td>
      <td>248913</td>
      <td>154055</td>
      <td>715.186268</td>
      <td>272.549602</td>
      <td>2.624059</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Niagara Frontier Transportation Authority</td>
      <td>Buffalo</td>
      <td>0</td>
      <td>DO</td>
      <td>30928</td>
      <td>82845</td>
      <td>51917</td>
      <td>771.541904</td>
      <td>288.034860</td>
      <td>2.678641</td>
    </tr>
    <tr>
      <th>12</th>
      <td>San Diego Metropolitan Transit System</td>
      <td>San Diego</td>
      <td>0</td>
      <td>DO</td>
      <td>173101</td>
      <td>490197</td>
      <td>317096</td>
      <td>476.443989</td>
      <td>168.244463</td>
      <td>2.831855</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Metro Transit</td>
      <td>Minneapolis</td>
      <td>0</td>
      <td>DO</td>
      <td>148150</td>
      <td>426665</td>
      <td>278515</td>
      <td>478.731650</td>
      <td>166.228995</td>
      <td>2.879953</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Denver Regional Transportation District</td>
      <td>Denver</td>
      <td>1</td>
      <td>DO</td>
      <td>274042</td>
      <td>793856</td>
      <td>519814</td>
      <td>420.304618</td>
      <td>145.090694</td>
      <td>2.896841</td>
    </tr>
  </tbody>
</table>
</div>

As the table demonstrates, some agencies are deploying light rail exclusively with one-car trains, while others *average* almost 3 cars per train.

If rail does indeed have scale advantages, then we would expect that the car revenue hour cost for systems with high load factors would tend to be lower. Certainly, regional variations in labor policy and business costs, along with other complicating factors such as geography, will also contribute to the variation in car revenue hours costs. But given that cost-competitiveness at scale is an oft-cited advantage of light rail, one would still expect additional cars to be more cost effective as trains get longer.

Let's check if the basic "rail scales" correlation exists by comparing the *Car Revenue Hour Opex* (y-axis) for each agency against its *Load Factor* (x-axis). We'll use the [Pearson correlation coefficient](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient) method.


```python
corr_df = rail_economics_df[['Car RevHr Opex', 'Load Factor']].copy()
print(corr_df.corr())
vis_x = rail_economics_df['Load Factor']
vis_y = rail_economics_df['Car RevHr Opex']
b, m = polyfit(vis_x, vis_y, 1)
plt.plot(vis_x, vis_y, '.')
plt.plot(vis_x, b + m * vis_x, '-')
plt.show()
```
|                    |Car RevHr Opex|      Load Factor |
|--------------------|--------------|------------------|
|Car RevHr Opex      |   1.000000   |         -0.456663|
| Load Factor        |   -0.456663  |         1.000000 |

![png](project-connect-o-and-m_21_1.png)


The calculation results and accompanying visualization show that there is indeed a negative correlation. By convention, most statisticians/analysts will describe a correlation of -0.46 as "moderate", with -0.51 being the threshold for a "strong" negative correlation. However, given the many other factors that contribute to costs, this is a solid finding in favor of the "rail scales" argument.

One challenge that the Project Connect team likely faced when deciding on an O&M cost calculation methodology was how to split costs between fixed operating system expenses that have to be paid before even the first train runs, the expenses for the first car of a train, and the expenses of each additional train car. By opting for a flat per car revenue hour measure, this discussion was simplified for policymakers. Unfortunately, as the national LRT vehicle revenue hour coefficient of variation demonstrates, the flat rate method would miss the actual reported rates for most systems by substantial amounts.

To arrive at a less risky O&M cost estimate with more nuanced cost allocation, we'll use [ordinary least squares linear regression](http://setosa.io/ev/ordinary-least-squares-regression/) as an exploratory tool in finding a more accurate cost distribution. The dependent variable (i.e. the outcome) in the regression will be the total operating expense for each agency LRT service type. We will use train revenue hours, extra car revenue hours, and the peer dummy variable as the independent variables (i.e. the outcome drivers).


```python
model = sm.OLS(rail_economics_df['2017 OPEX'],
               sm.add_constant(rail_economics_df[['Train Revenue Hrs', 'Extra Car Revenue Hrs', 'Peer']]))
results = model.fit()
summary = results.summary()
print(summary.tables[0])
print(summary.tables[1])
```

<pre>

                                OLS Regression Results
    ==============================================================================
    Dep. Variable:              2017 OPEX   R-squared:                       0.754
    Model:                            OLS   Adj. R-squared:                  0.713
    Method:                 Least Squares   F-statistic:                     18.40
    Date:                Wed, 13 Nov 2019   Prob (F-statistic):           1.02e-05
    Time:                        12:05:43   Log-Likelihood:                -416.55
    No. Observations:                  22   AIC:                             841.1
    Df Residuals:                      18   BIC:                             845.5
    Df Model:                           3
    Covariance Type:            nonrobust
    ==============================================================================
    =========================================================================================
                                coef    std err          t      P>|t|      [0.025      0.975]
    -----------------------------------------------------------------------------------------
    const                 -2.694e+06   1.77e+07     -0.152      0.881   -3.99e+07    3.45e+07
    Train Revenue Hrs       511.8985    124.499      4.112      0.001     250.337     773.460
    Extra Car Revenue Hrs   117.5641    100.635      1.168      0.258     -93.863     328.991
    Peer                  -3.182e+07   2.16e+07     -1.470      0.159   -7.73e+07    1.37e+07
    =========================================================================================
</pre>

The regression results are promising but not straightforward for the typical Austin resident or policymaker to interpret.

The [R-squared](https://www.khanacademy.org/math/ap-statistics/bivariate-data-ap/assessing-fit-least-squares-regression/a/r-squared-intuition) is strong at 0.754, yet that is unsurprising given that anyone with real-world understanding of rail economics would expect train and car revenue hours to be clear and substantial contributors to overall operating expense level.

Only the p-value for *Train Revenue Hours* meets the conventional threshold for statistical significance. Additional train cars are priced at 23% the hourly rate of the first train car, though the p-value is not low enough to be confident about that specific discount factor.

There is a \$31.8 million (2017) discount to the total annual operating expense for peer cities relative to non-peer cities. This makes sense given the previous discussion of lower peer city LRT costs, though again, the p-value is a tad too high to confidently use this specific coefficient's effect size.

The author can not calculate the estimated costs for the Project Connect LRT options using the regression coefficients because the existing O&M memos do not break out the train and passenger car hours.

Overall, the regression results award another round to the conventional view that "rail scales" over Project Connect's flat rate simplification. The regression results also bolster the case for using a subset of peer cities instead of the national average.

During the 2013-2014 version of Project Connect, its [main O&M memo](https://keepaustinwonky.files.wordpress.com/2014/09/centralcorridorhct-omcostestimate140619-june-202c-2014.pdf) featured a LRT opex estimation approach that avoided a flat rate in favor of mix of fixed system costs, train costs, and extra car costs. That previous methodology also selected a set of peer cities (Seattle, Charlotte, Minneapolis, Houston, Phoenix, Hampton Road) instead of the entire national sample. Interestingly enough, the share of total opex that was assigned to passenger car-specific expenses under that model was 19%. Compare that to how the regression results price the extra passenger car revenue hour rate at 23% of the first-car train setup. The similarity is worth noting. And it would be valuable for present-day Project Connect to explain why its flat rate is a superior approach than the 2013-2014 technique.

### Orange-Blue LRT under "Rail Scales"

There is a significant, troublesome implication for Project Connect's choice to use a flat passenger car rate for estimating O&M costs instead of a mix of costs reflecting a "rail scales" methodology.

A flat rate makes a LRT with a lower load factor close the productivity gap with a LRT with a higher load factor.

To illustrate this, we compare Project Connect's estimated national LRT revenue hour cost under several train configurations against the regression results above.


```python
vendor_train_17 = 284.15
review_train_17 = 511.90
vendor_car_17 = 284.15
review_car_17 = 117.56
calculations = {
    'Configuration': ['1 Car Train', '2 Car Train', '3 Car Train'],
    'Project Connect': [vendor_train_17, (vendor_train_17 + vendor_car_17), (vendor_train_17 + (vendor_car_17 * 2))],
    'Informatx': [review_train_17, (review_train_17 + review_car_17), (review_train_17 + (review_car_17 * 2))]
}
pd.DataFrame(data=calculations)
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Configuration</th>
      <th>Project Connect</th>
      <th>Informatx</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1 Car Train</td>
      <td>284.15</td>
      <td>511.90</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2 Car Train</td>
      <td>568.30</td>
      <td>629.46</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3 Car Train</td>
      <td>852.45</td>
      <td>747.02</td>
    </tr>
  </tbody>
</table>
</div>



The table above details the different cost curves for each method. As the reader can see, Project Connect's flat rate makes a low load factor service profile seem more efficient than it would be scored taking rail's economies of scale into consideration. The complement of that is that LRT service profiles with high load factor are denied the cost benefits of scale.

For the sample 2028 schedule created by Project Connect, the Blue Line LRT runs single-car trains for an overwhelming plurality of its time slots. By using a flat rate, Project Connect makes Blue Line LRT more cost-competitive against Blue Line BRT (and Orange Line LRT) than it would be if rail's economies of scale affected the cost curve.

The Orange Line's 2028 sample LRT schedule has a plurality of multi-car trains for its time slots, with weekday workday hours deploying 3-car trains. Using a flat rate approach denies the Orange Line LRT the cost advantages of scale, making it seem more expensive relative to BRT and Blue Line LRT.

## Recommendations

The issues concerning Project Connect's O&M estimates raised in this Notebook are serious enough that media, advocates, policymakers, and Project Connect staff themselves should use terms such as "draft" and "preliminary" when referring to them.

Second, Project Connect should provide greater detail about the origins of its BRT revenue hour cost figure, as well as why the technique that was used is capable of providing an accurate estimate.

Third, Project Connect should explain its shift from a peer-city-based, scale-aware LRT revenue hour cost estimation method during its 2013-2014 vintage to the national flat rate used today.

Fourth, Project Connect should produce a new set of cost documents that add the national average BRT revenue hour to BRT O&M cost prediction tables, as well as a peer-city-based, scale-aware LRT revenue hour cost estimate to LRT tables.

*Informatx's Jupyter notebooks are available on [Github](https://github.com/jga/informatx-notebooks).*
