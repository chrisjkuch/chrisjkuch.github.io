---
layout: post
title:  "Northwestern Bus Data - Take 2 (with pandas)"
date:   2017-04-24 16:00:00 -0500
categories: jekyll evanston
---

After yet another frigid spent waiting for a late bus, my flustered wife vented via text about the unreliability of Northwestern's intercampus shuttle. Armed with a little bit of code and a matrimonial sense of duty, I attempted to determine conclusively the shuttle's reliability.

The data analyzed in this notebook was collected in August-October 2015, about 6 weeks (~30 stops per schedule stop) worth in total. We'll use heatmaps to visualize whether the shuttle is on time and by how much it deviates from its schedule.


```python
# Let's import everything we need and set some parameters
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib
import seaborn as sns
%matplotlib inline
plt.style.use('ggplot')
matplotlib.rcParams.update({'font.size': 16})
matplotlib.rcParams.update({'xtick.labelsize': 14})
matplotlib.rcParams.update({'ytick.labelsize': 14})
```

First, let's read in the data! The data about when each shuttle stop was made is in `intercampus_data.csv`. The northbound and southbound schedules are in `evanstontochicago.csv` and `chicagotoevanston.csv`.


```python
stops = pd.read_csv('intercampus_data.csv')
northsouth = pd.read_csv('evanstontochicago.csv')
southnorth = pd.read_csv('chicagotoevanston.csv')
```

What do we have here...


```python
stops.head(5)
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Route ID</th>
      <th>Route Name</th>
      <th>Bus ID</th>
      <th>Bus Name</th>
      <th>Stop ID</th>
      <th>Stop Name</th>
      <th>Seconds after midnight</th>
      <th>Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>48</td>
      <td>Intercampus (I)</td>
      <td>29974</td>
      <td>29741</td>
      <td>294</td>
      <td>Emerson/Maple</td>
      <td>64653</td>
      <td>2016-08-08 17:57:33.504574</td>
    </tr>
    <tr>
      <th>1</th>
      <td>48</td>
      <td>Intercampus (I)</td>
      <td>29729</td>
      <td>29729</td>
      <td>282</td>
      <td>Central L Station (westbound)</td>
      <td>64674</td>
      <td>2016-08-08 17:57:54.206635</td>
    </tr>
    <tr>
      <th>2</th>
      <td>48</td>
      <td>Intercampus (I)</td>
      <td>29727</td>
      <td>29727</td>
      <td>313</td>
      <td>Chicago/Kedzie</td>
      <td>64694</td>
      <td>2016-08-08 17:58:14.943511</td>
    </tr>
    <tr>
      <th>3</th>
      <td>48</td>
      <td>Intercampus (I)</td>
      <td>29729</td>
      <td>29729</td>
      <td>252</td>
      <td>Central/Jackson (westbound)</td>
      <td>64745</td>
      <td>2016-08-08 17:59:05.908323</td>
    </tr>
    <tr>
      <th>4</th>
      <td>48</td>
      <td>Intercampus (I)</td>
      <td>29974</td>
      <td>29741</td>
      <td>183</td>
      <td>Sherman/Emerson</td>
      <td>64766</td>
      <td>2016-08-08 17:59:26.634276</td>
    </tr>
  </tbody>
</table>
</div>



Ah, it appears we have some information about the route, the bus, the stop, and when the bus on a given route stopped at a given stop. That last piece of information appears in two formats: as a timestamp and as the number of seconds after midnight; the latter will be a little bit easier to work with because we want to overlay all of the stop times on the same 1-day timescale.

Let's quickly plot some arbitrary data over time to see what it looks like.


```python
stops['Time'] = pd.to_datetime(stops['Time'])
stops.plot(x='Time', y='Seconds after midnight', figsize=(10, 6))
plt.ylabel('Seconds after midnight')
```




    <matplotlib.text.Text at 0x7f5db1000cf8>




![png](/assets/bus_v2/Bus_Plotter_pd_7_1.png)


So we can see that there are some chunks of data missing, but it looks like we have about 5-6 weeks worth of data that should correspond to about 25-30 recorded stops per scheduled shuttle stop.

Here are all the stops on our route:


```python
pd.unique(stops['Stop Name'])
```




    array(['Emerson/Maple', 'Central L Station (westbound)', 'Chicago/Kedzie',
           'Central/Jackson (westbound)', 'Sherman/Emerson', 'Sherman/Foster',
           'Chicago/Lee', 'Ryan Field', 'Sherman/Gaffield',
           'Sheridan/Loyola (southbound)', 'Weber Arch', 'Jacobs Center',
           'Chicago/Sheridan', 'Tech Institute', 'Patten Gym', 'Chicago/Grove',
           'Central/Jackson (eastbound)', 'Central L Station (eastbound)',
           'Noyes L Station', 'Chicago/Greenleaf', 'Ridge/Civic Center',
           'Chicago/Main', 'Ridge/Reserve Apartments', 'Ward Building',
           'Sheridan/Noyes', 'Chicago/Davis', 'Sheridan/Foster',
           'Sheridan/Loyola (northbound)'], dtype=object)



Now let's look at the schedule data. 


```python
northsouth.head(5)
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Ryan Field</th>
      <th>Central/Jackson (eastbound)</th>
      <th>Central L Station (eastbound)</th>
      <th>Noyes L Station</th>
      <th>Ridge/Civic Center</th>
      <th>Ridge/Reserve Apartments</th>
      <th>Emerson/Maple</th>
      <th>Sherman/Emerson</th>
      <th>Sherman/Foster</th>
      <th>Sherman/Gaffield</th>
      <th>Sheridan/Noyes</th>
      <th>Sheridan/Foster</th>
      <th>Chicago/Sheridan</th>
      <th>Chicago/Grove</th>
      <th>Chicago/Greenleaf</th>
      <th>Chicago/Main</th>
      <th>Sheridan/Loyola (southbound)</th>
      <th>Ward Building</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.274329</td>
      <td>0.275023</td>
      <td>0.275718</td>
      <td>0.277106</td>
      <td>0.278495</td>
      <td>0.279190</td>
      <td>0.280579</td>
      <td>0.281968</td>
      <td>0.282662</td>
      <td>0.283356</td>
      <td>0.284745</td>
      <td>0.286134</td>
      <td>0.287523</td>
      <td>0.289606</td>
      <td>0.291690</td>
      <td>0.292384</td>
      <td>0.299329</td>
      <td>0.327106</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.284745</td>
      <td>0.285440</td>
      <td>0.286134</td>
      <td>0.287523</td>
      <td>0.288912</td>
      <td>0.289606</td>
      <td>0.290995</td>
      <td>0.292384</td>
      <td>0.293079</td>
      <td>0.293773</td>
      <td>0.295162</td>
      <td>0.296551</td>
      <td>0.297940</td>
      <td>0.300023</td>
      <td>0.302106</td>
      <td>0.302801</td>
      <td>0.309745</td>
      <td>0.337523</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.295162</td>
      <td>0.295856</td>
      <td>0.296551</td>
      <td>0.297940</td>
      <td>0.299329</td>
      <td>0.300023</td>
      <td>0.301412</td>
      <td>0.302801</td>
      <td>0.303495</td>
      <td>0.304190</td>
      <td>0.305579</td>
      <td>0.306968</td>
      <td>0.308356</td>
      <td>0.310440</td>
      <td>0.312523</td>
      <td>0.313218</td>
      <td>0.320162</td>
      <td>0.347940</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.315995</td>
      <td>0.316690</td>
      <td>0.317384</td>
      <td>0.318773</td>
      <td>0.320162</td>
      <td>0.320856</td>
      <td>0.322245</td>
      <td>0.323634</td>
      <td>0.324329</td>
      <td>0.325023</td>
      <td>0.326412</td>
      <td>0.327801</td>
      <td>0.329190</td>
      <td>0.331273</td>
      <td>0.333356</td>
      <td>0.334051</td>
      <td>0.340995</td>
      <td>0.368773</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.347245</td>
      <td>0.347940</td>
      <td>0.348634</td>
      <td>0.350023</td>
      <td>0.351412</td>
      <td>0.352106</td>
      <td>0.353495</td>
      <td>0.354884</td>
      <td>0.355579</td>
      <td>0.356273</td>
      <td>0.357662</td>
      <td>0.359051</td>
      <td>0.360440</td>
      <td>0.362523</td>
      <td>0.364606</td>
      <td>0.365301</td>
      <td>0.372245</td>
      <td>0.393079</td>
    </tr>
  </tbody>
</table>
</div>



The format for this data appears to be the day-fraction at which the bus stops. To use it, we'll just multiply by 24 x 60 to get units of minutes or 24 x 3600 to get units of seconds.


```python
# Convert scheduled times into units of minutes
northsouth = northsouth * 24 * 60
southnorth = southnorth * 24 * 60
```


```python
pd.unique(northsouth.columns)
```




    array(['Ryan Field', 'Central/Jackson (eastbound)',
           'Central L Station (eastbound)', 'Noyes L Station',
           'Ridge/Civic Center', 'Ridge/Reserve Apartments', 'Emerson/Maple',
           'Sherman/Emerson', 'Sherman/Foster', 'Sherman/Gaffield',
           'Sheridan/Noyes', 'Sheridan/Foster', 'Chicago/Sheridan',
           'Chicago/Grove', 'Chicago/Greenleaf', 'Chicago/Main',
           'Sheridan/Loyola (southbound)', 'Ward Building'], dtype=object)



Note that not all of the stops are on each route! Most are northbound or southbound only. 

First, let's look at our morning route from Evanston to Chicago by collecting a list of each stop's recorded shuttle stops using the unit of seconds after midnight.


```python
# Collect recorded stops in units of minutes
nsstop_times = {} # Dictionary of recorded stop times 
for stop_name in northsouth.columns:
    nsstop_times[stop_name] = stops[stops['Stop Name'] == stop_name]['Seconds after midnight'] / 60
snstop_times = {} # Dictionary of recorded stop times 
for stop_name in southnorth.columns:
    snstop_times[stop_name] = stops[stops['Stop Name'] == stop_name]['Seconds after midnight'] / 60
    
# Parameters to control the window
start_hour = 6
end_hour = 24
interval_in_min = 1

# Make the bins and the histogram
interval_bins = np.arange(start_hour * 60, end_hour * 60 + 1, interval_in_min)
nsstop_times['Sherman/Foster'].hist(bins=interval_bins, figsize=(12,4), color='b')

# Formatting
time_labels = [ str(x) + ':00' for x in range(start_hour, end_hour)]
plt.xlim(start_hour * 60, end_hour * 60)
plt.xticks(np.arange(start_hour * 60, end_hour * 60, 60), time_labels)
plt.xlabel('Time')
plt.ylabel('Number of stops recorded')
plt.show()
```


![png](/assets/bus_v2/Bus_Plotter_pd_17_0.png)


We're looking at a plot of the number of stops recorded in 1-minute bins during the day. Already we can see a few interesting trends - large peaks should probably correspond with regularly scheduled stop times. Let's see where the shuttle stops actually are by writing a function and using it to look more closely at the data.


```python
def plot_stop_histogram(stop_series, scheduled_stops, start_hour=6, end_hour=24, minutes_per_interval=1):
    '''Plots a histogram of recorded shuttle stops in time bins
        input
            stop_series - Pandas Series of recorded stop times in units of seconds after midnight
            scheduled_stops - Pandas Series of scheduled stops
            start_hour - hour to start binning
            end_hour - hour to stop binning
            minutes_per_interval - how many minutes each bin should encapsulate
        output
            None
    '''
    # Create the bins for the histogram
    interval_bins = np.arange(start_hour * 60, end_hour * 60 + 1, minutes_per_interval)
    stop_series.hist(bins=interval_bins, figsize=(12,4), color='b', alpha=1, grid=False)
    time_labels = [ str(x) + ':00' for x in range(start_hour, end_hour)]
    
    # Format the plot
    plt.xlim(start_hour * 60, end_hour * 60)
    plt.xticks(np.arange(start_hour * 60, end_hour * 60, 60), time_labels)
    plt.xlabel('Time')
    plt.ylabel('Number of stops recorded')
    
    # Add vertical lines at scheduled stop times, remembering to convert the scheduled time from day to minute
    mincts, maxcts = plt.ylim()
    plt.vlines(scheduled_stops, ymin=0, ymax=maxcts, linestyles='--', color='k', alpha=0.2)
    plt.show()
```

I live at Sherman and Foster, so let's look there and zoom in on the evening rush hour.


```python
stop_name = 'Sherman/Foster'
plot_stop_histogram(nsstop_times[stop_name], northsouth[stop_name], start_hour=6, end_hour=23)
plot_stop_histogram(nsstop_times[stop_name], northsouth[stop_name], start_hour=15, end_hour=20)

```


![png](/assets/bus_v2/Bus_Plotter_pd_21_0.png)



![png](/assets/bus_v2/Bus_Plotter_pd_21_1.png)


Uh oh. Looks like we've got some differences at rush hour between the scheduled stops (dashed lines) and the recorded stops (blue histogram). Let's visualize this another way using a heatmap. This will let us visualize the frequency at which stops occurred in a certain window and by choosing an appropriate color scheme will tell us whether the stop was made at the appropriate time.


```python
# Get the histogram in numerical form
stop_name = 'Sherman/Foster'
scheduled_stops = northsouth[stop_name].astype(int)
start_hour = 15
end_hour = 20
time_labels = [ str(x) + ':00' for x in range(start_hour, end_hour)]
# 1-minute intervals
interval_bins = np.arange(start_hour * 60, end_hour * 60 + 1, 1)
counts, hist_bins = np.histogram(nsstop_times[stop_name], bins=interval_bins)

# Apply a mask using a window of time
window = 3
mask = -1 * np.ones_like(counts)
for scheduled_stop in scheduled_stops:
    index = scheduled_stop - start_hour * 60
    mask[index-window:index+window] = 1

maxcounts = max(counts)
counts = counts * mask
counts = counts.reshape(1, counts.size)
plt.figure(figsize=(15,2))
hmax = sns.heatmap(counts, cmap='RdBu', yticklabels=False, vmin=-maxcounts, vmax=maxcounts)
hmax.collections[0].colorbar.set_ticks([-maxcounts, -maxcounts/2.5, maxcounts/2.5, maxcounts])
hmax.collections[0].colorbar.set_ticklabels(['Frequently late', 'Less-frequently late', 'Less frequently on-time', 'Frequently on-time'])
plt.xticks(np.arange(0, (end_hour - start_hour) * 60, 60), time_labels)
plt.xlabel('Time')
plt.ylabel('Sherman/Foster')
plt.show()
```


![png](/assets/bus_v2/Bus_Plotter_pd_23_0.png)


What are we looking at here? Well, each line represents a one-minute slice of time, and the color of that line indicates whether that slice of time is within some window of time (in this case, 3 minutes) of a scheduled shuttle stop. Blue is on time, red is late, and the intensity of the color is proportional to the number of stops made within a one-minute window. 

As a result, a dark blue line should indicate that the shuttle frequently stops at the scheduled time, while a dark red line indicates it reliably stops *at a time at least 3 minutes outside* the scheduled time. Furthermore, the lines with lighter coloring (less intensity) show that while shuttle stops were recorded during that window of time, the frequency of stops occuring in the 1-minute window is low. Lots of light lines stacked together (of either color) indicate a high variance in shuttle arrival time.

In short, dark blue lines are good and anything else is problematic. 

So the 3pm shuttle stop looks like we're mostly in good shape. However, starting at 4pm, things start to break down. 


```python
def bus_heatmap(recstops_dict, schedule_df, start_hour=6, end_hour=23, time_window=5, intvl=1):
    '''Create the heatmap for our entire schedule'''
    stop_names = schedule_df.columns.tolist()
    recstops_2d = np.zeros((len(stop_names), (end_hour - start_hour) * 60))
    intvl_bins = np.arange(start_hour * 60, end_hour * 60 + intvl, intvl)
    
    def apply_window(stops_hist, targets, window):
        '''Apply +1 multiplier to stop times in stops within window of targets'''
        mask = -1 * np.ones_like(stops_hist)
        for scheduled_stop in targets:
            index = (scheduled_stop - start_hour * 60) // intvl
            mask[index - window // intvl:index + window // intvl] = 1
        counts = stops_hist * mask
        counts = counts.reshape(1, counts.size)
        return counts
    
    # Get scheduled & recorded stop times for each stop
    for i, name in enumerate(stop_names):
        scheduled_stops = schedule_df[name]
        stop_hist, bins = np.histogram(recstops_dict[name], bins=intvl_bins)
        recstops_2d[i, :] = apply_window(stop_hist, scheduled_stops, time_window)
    
    maxcounts = np.amax(recstops_2d)
    plt.figure(figsize=(15,10))
    hmax = sns.heatmap(recstops_2d, cmap='RdBu', yticklabels=stop_names, vmin=-maxcounts, vmax=maxcounts)
    hmax.collections[0].colorbar.set_ticks([-maxcounts, -maxcounts/2.5, maxcounts/2.5, maxcounts])
    hmax.collections[0].colorbar.set_ticklabels(['Frequently late', 'Less-frequently late', 'Less frequently on-time', 'Frequently on-time'])
    time_labels = [ str(x) + ':00' for x in range(start_hour, end_hour)]
    plt.xticks(np.arange(0, (end_hour - start_hour) * 60, 60), time_labels)
    plt.xlabel('Time')
    plt.show()
    
```


```python
bus_heatmap(nsstop_times, northsouth)
```

    /home/chris/anaconda3/lib/python3.5/site-packages/ipykernel/__main__.py:12: VisibleDeprecationWarning: using a non-integer number instead of an integer will result in an error in the future



![png](/assets/bus_v2/Bus_Plotter_pd_26_1.png)


What does this complicated chart mean? Basically, there are three periods during the day in which the bus is almost *never* on time: the stop leaving Ryan field at ~9:00am, ~12:00pm, and between ~5:30 and 9:00pm. The window applied to this data is 5 minutes on either side of the window in which the bus is scheduled to arrive, meaning that an arrival outside this window means that if you wait for 5 minutes before the bus arrives for any one of these stops, you could expect to wait for more than 10 minutes before a bus arrives to pick you up! Furthermore, the consistency with which certain late stops are made suggests that it is not necessarily a problem of predicatbility but rather simply that the schedule should be modified to reflect the reality even if nothing changes on the back end.


```python
bus_heatmap(snstop_times, southnorth)
```

    /home/chris/anaconda3/lib/python3.5/site-packages/ipykernel/__main__.py:12: VisibleDeprecationWarning: using a non-integer number instead of an integer will result in an error in the future



![png](/assets/bus_v2/Bus_Plotter_pd_28_1.png)



```python
bus_heatmap(nsstop_times, northsouth, start_hour=15, end_hour=21)
```

    /home/chris/anaconda3/lib/python3.5/site-packages/ipykernel/__main__.py:12: VisibleDeprecationWarning: using a non-integer number instead of an integer will result in an error in the future



![png](/assets/bus_v2/Bus_Plotter_pd_29_1.png)



```python
bus_heatmap(snstop_times, southnorth, start_hour=15, end_hour=21)
```

    /home/chris/anaconda3/lib/python3.5/site-packages/ipykernel/__main__.py:12: VisibleDeprecationWarning: using a non-integer number instead of an integer will result in an error in the future



![png](/assets/bus_v2/Bus_Plotter_pd_30_1.png)



```python

```
