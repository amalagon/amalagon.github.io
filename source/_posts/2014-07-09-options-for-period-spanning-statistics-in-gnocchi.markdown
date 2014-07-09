---
layout: post
title: "Options for Period-Spanning Statistics in Gnocchi"
date: 2014-07-09 09:06:40 -0400
comments: true
categories: [OPW, Openstack] 
---
Options for Period-Spanning Statistics in Gnocchi:

We would like to implement aggregation functions that span periods such as simple moving averages or exponentially weighted moving averages (EWMA). See the “Functions Defined” section at the end for descriptions of EWMA and Moving Averages. This is getting kind of long, so it’s probably enough to read A., B., Summary, and Functions Defined sections.

There are a few different options for implementing this, depending on whether Gnocchi ends up retaining raw data points or just doing roll-up, and how you deal with the prior history when calculating the period-spanning statistic. For the latter, there are mainly two strategies:

A. No prior history

Start at the the timestamp specified by the user, and aggregate forward in time. This will produce a lag between the aggregate and the data only for the moving average function; e.g. if the window size is defined as covering  7 periods, the moving average will first give a result for the 4th period  (if you define the moving average to be centered). EDIT: Exponential smoothing does not produce a lag, even if  the input parameter spanis larger than the array to be smoothed. However, the first element of the output EWMA is always the same as the first element input array, so your first period will have no smoothing applied. Also if span > len(inputarray),  your smooth is usually a bad fit to the data.

B. Look-back

Incorporate the values from k bins before the start timestamp specified by the user, where 2k+1 is the window size of the moving average; for EWMA look back to span bins. The exponentially weighted moving average command in Pandas takes an input parameter of span, and double exponential smoothing (Holt-Winters) takes input parameters span and beta. For both cases we can relate span to the usual input parameter a via span=2-a.

Note: For the moving average statistic, if we only retain the aggregates and not raw data, we need to know the number of data points in each period (ie count needs to be available). If we retain all the data points, this is not necessary.

Static vs Free Parameters:

We can either define the window size/span (and for Holt-Winters) when we create the entity, or leave them as parameters that get specified in each query.

My picture of Gnocchi is that timestamps get acquired in time-order for the most part - basically you are usually getting more recent timestamps. For this model and for static parameters, you can calculate EWMA/MovA values as you fill up the period buckets. Originally, the idea was that even if periods expired, we could keep “ghostly remnants” (EWMA/MovA values) from the expired bins if needed to calculate new EWMA/MovA values for current bins, but this is unnecessary (I’ll ramble about why for the next few paragraphs).

Since EWMA only depends on the last forecast to make the next prediction, you generally only need to keep the EWMA values for two periods at a time - one EWMA value that gets updated as you add more timestamps to the current period, and an EWMA value that is fixed from the previous period, where no new timestamps are being added, but that is used to predict the EWMA of the current period. EDIT: For systems where there are n > 1 periods that are still being updated, you need to keep all those periods plus the first period that is locked-down (no new datapoints expected to arrive). For moving averages you do need to retain the last k average values of the periods before the start timestamp as well as the count for those periods. However, the moving average is not a recursive function, so you don’t actually depend on the moving average in previous periods to calculate the moving average in the current period. The only additional feature required to do the moving average that is not currently in Gnocchi would be saving the count. This is of course assuming we’re only saving aggregates - if we retain all datapoints, then you must only save the last k timestamp/value pairs.

In short, you don’t have to retain many EWMA/MovA values from old periods in order to calculate them for current periods (at most need to retain 1 old EWMA value for EWMA, don’t need any old MovA values for MovA, although you need to retain the mean and count of the k previous bins). If we assume k is less than the number of periods we generally retain, this means we never have to worry about needing to keep statistics from expired periods.

However, if you get some out of order timestamp that belongs in an older period, this will affect the moving average value for periods within window_size of the bin that holds the timestamp. The effect on EWMA values is more pervasive - probably negligible for for periods > span away from the re-updated bin. In this case you need to keep EWMA values from periods before the modified bucket in order to recalculate the EWMA. If your data is coming in in a completely non-time ordered way, there is no sense in the above scheme for calculating the EWMA/MovA values for bins as you go, since you’ll just have to recalculate everything as new points come in for old time buckets. For that case, you should use queries.
 
Queries: 
Calculate the aggregation on demand without storing partial results. If we define the window size as the number of periods it spans, a query would look something like:

	  GET /v1/entity/<UUID>/measures?start='2014-06-21 23:23:20’&stop='2014-06-22 23:23:20’&aggregation=ewma&span=20

or

	GET /v1/entity/<UUID>/measures?start='2014-06-21 23:23:20’&stop='2014-06-22 23:23:20’&aggregation=mva&window=5

or 

   GET /v1/entity/<UUID>/measures?start='2014-06-21 23:23:20’&stop='2014-06-22 23:23:20’&aggregation=holtwinters&span=5&beta=.1

Adding scoped names might be useful as well (ie aggregation.window, aggregation.beta)

Handling boundaries: If you specify a start timestamp in the query that’s expired, start from the data that does exist. If you specify a start timestamp that needs data points from expired bins to calculate MovA/EWMA (basically you chose the oldest non-expired period), choose Strategy A of no prior history, and aggregate starting from there to the stop timestamp. Strategy B doesn’t really work here since you don’t know how many bins to retain as spanand k are not defined until the query is proposed. Another way would be to not allow the user to request Moving Average on timestamps unless the the window size is such that you don’t run into expired bins (or truncate the window size so that condition is met). If we count the oldest non-expired bin as b0 and if the start timestamp in the query t_i is in bin b_i, the condition would be that the window-size < 2*i. For EWMA since it returns values no matter what size the span is, probably going with Strategy A is easiest.

We could also implement a combination of both static parameters and query functionality. That is, specify window size, span, and beta upfront when creating the entity. Then if you later want say a different window size, do a query. 

Summary:

If your data is not time ordered, you should only implement query functionality.

If your data is time ordered, you can implement static parameters (meaning you define window size/span/beta upon entity creation). Using static parameters avoids the problem of needing data from bins that have expired.

Whatever you decide to implement, be explicit in the documentation about whether Moving-Average[value, window_size] is centered on the value or not. Centering makes more sense to me conceptually, but up to the user.

In general, Strategy A of no prior history is simpler than Strategy B of taking old bins into account. In Strategy B you have to do more checking to see if the old bins aren’t expired, etc. Probably easiest to do Strategy A.

Right now the period-spanning statistics need the mean and count of each period. Since you can re-adjust the mean of a bin when you add new points in that time bucket by doing (old-mean*old-count+new-value)/(old-count+1)
really what Moving Average is doing to calculate the moving average over 3 periods is:
MovingAverage(3 bins)=(mean1*count1+mean2*count2+mean3*count3)/(count1+count2+count3)
so you don’t have to retain all data points, just the mean and count (which can be updated correctly even if new timepoints come in for old buckets).

of course if you retained all the data points, it would be eqiuvalent to do
MovingAverage(3 bins) = Sum(values in 3 bins wide interval)/Number of points

Notice that for count1=count2=count3the moving average is equal to the average of the averages:
MovingAverage(3bins) = (mean1+mean2+mean3)/3

In the end, it comes down to what the data is expected to look like. If you expect almost uniform sampling, then taking the average of averages should be good enough. If however, you expect 300 bins in one period and 2 in another and hugely noisy data...I can’t say for sure that the average of averages is worthless, but I think in most cases it would be a worse estimate of the mean of the k bins than when weighting by count.

The EWMA is a bit trickier. Technically what we are doing here is an exponential smooth over the mean values of the bins (if all we retain are the mean/count). E.g.:

EWMA(1stbin)=mean1
EWMA(2ndbin)=(1-)EWMA(1stbin)+*mean2
etc...
where =2/(1+span)

In order to do an EWMA on the raw data, you do need all the data points to be retained.

Summary of the Summary:

The question seems to be, do we retain all data points or not?

Right now, you can do period-spanning statistics on the aggregated-data (where we’re only looking at period-spanning averages). It requires some work, since you have to be careful about old data points changing the aggregated mean value in old periods, and really you’re only doing the EWMA on the means, not the actual raw data. Also, the window size/span are defined as the number of periods to span, not the number of raw data points, which could be confusing. However, you’re not storing all the data points, which seems to be the intention.

If you save all the data points, doing period-spanning statistics is conceptually more straightforward. You don’t have to define the window size in terms of periods, and EWMA can be done on the raw data. Moreover, you can do other period-spanning statistics that were just impossible with roll-up (period-spanning stddevs, etc).  However, you’re now retaining all the data points, which seems to be something Gnocchi was trying to avoid.
Functions Defined:

Just to define what we mean here, the output of a moving average reads:
y[n] = 1/(2k+1)*sum(x[n-k:n+k+1]) 

where the odd number 2k+1is the window size and x[t]the input array (in our case, each element in x corresponds to the mean*count of a period). Increasing the window size increases the smoothing effect.. In Pandas the moving average syntax is:
y = pandas.rolling_mean(x, window_size, center=True)

If we retain all datapoints, we can call pd.rolling_mean on the raw data. If we are doing roll-up, you can’t use this syntax, but need to implement a moving_average that takes as input arrays mean, count corresponding to the periods being looked at:

moving_average(window_size=# of periods to aggregate over, mean, count)

The output of an exponentially weighted moving average is:

y[t] = x[t]+(1-)y[t-1], t>0
y[0]=x[0]
y = pandas.ewma(x, com= (1-alpha)/alpha)
or
y = pandas.ewma(x, span = (2-alpha)/alpha)

As pandas.ewma takes in spanas a parameter, rather than , it is perhaps clearer to say that =2/(1+span).is a smoothing parameter between 0 and 1; for small , weights decay quickly for old data points and they “fall off” in affecting the forecast. For large , weights decay more slowly and many data points contribute to the forecast.
