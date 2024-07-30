# BRIL Luminosity Anomaly Detection Algorithm
The code provided in the anomaly_detection.ipynb code in this repository is designed for individual anomaly selection of the PLT channels at CMS and has been further extended to perform a more general channel selection of PLT data. In the near future a version designed for the BCM1F detector at CMS will perform exactly the same way but is designed to handle 48 channels of data rather than 16 channels of data. 

This code isn't ML based, so no training of a model is required. The python package adtk must be installed to ensure the code runs, however. In my jupyter notebook I have the first cell dedicated in the code to ensuring that adtk is installed and located in the correct python path, which is very useful if running on CERN servers and using the SWAN interface. If you are running through a terminal, simply ensure adtk is intalled prior by running the following:

```
!pip install adtk
```

Ensure that when loading in the data it is an hd5 file. If the columns need to be renamed, ensure they are. For the PLT based code, the columns that are important are the individual plt channels - currently named ```pltagg_columns = [f'pltaggzero_{i}' for i in range(16)]```, the date-time column, and the average HF and BCM1F data columns, respectively named ```'hfetlumi_avgraw'``` and ```'bcm1flumi_avgraw'```.

One thing this code relies upon is comparing all channels to a reference. We choose this reference by comparing all channels that are known to be good in a particular fill, calculate the median point for every point across all channels, and then compare the actual channels against this median channel; whichever channel is closest overall is therefore the reference, which is used later on in the code. The code for this is listed below and also found in the python notebook: 
```
channels_failed = [] 
channels_good = [ch for ch in channels if ch not in channels_failed]
all_lumi_values = []
for ch in channels_good:
    all_lumi_values.extend(df_data[ch]['lumi'].values)
median_channel_series = pd.Series(all_lumi_values).median()
channel_median_diff = {}
for ch in channels_good:
    difference_plt_median = np.abs(df_data[ch]['lumi'] - median_channel_series)
    median_diff = difference_plt_median.median()
    channel_median_diff[ch] = median_diff 
closest_channel = min(channel_median_diff, key=channel_median_diff.get)
print(f"The reference channel for fill {fill_number} is Channel {closest_channel}.")
reference = [closest_channel]
```

There are a few ways to perform anomaly detection in this code. The first, marked by the heading cell ```Begin Channel Selection``` performs a more widespread anomaly detection model that checks to see if the channels have passed or have failed. We take into account a few different conditions for a channel to be flagged as bad. First, we check the total number of final anomalies flagged, which varies based on how spread out the luminosity measurements are for a particular fill - ie, a fill that ranges over 20,000 luminosity when plotted will be treated differently than a fill that ranges over 1,000 luminosity. If enough anomalies are flagged, we then check the distribution of those anomalies to see how spread out or compact in a region they are detected. 

Two different situations are taken into account when finding anomalies: the ratio plot and the difference plot. When checking for anomalies in PLT, BCM1F, or HF data we need to ensure we don't accidentally flag the emittance scan as an anomaly - since by eye it looks like an anomaly that extends across all channels. Thus, we try to eliminate the emittance scan by taking the difference between the channel we're checking and the reference channel and the ratio between the selected channel and the reference channel. If a channel fails the channel selection, then we move on to using the ```full_automated``` function. This displays how the anomalies are distributed and highlights what threshold failed causing the channel to be marked as bad. From here the user can determine if the channel is actually okay and just barely failed or if some portion/all of the data needs to be removed.

Finally, after performing channel selection, make sure to append the data to the csv files by running the function ```process_fill``` and run the cells following to visualize the distribution and information pertaining to all runs you have previously examined.

Et voila! You have performed channel and/or anomaly detection for PLT and BCM1F data! 
