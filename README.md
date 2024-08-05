# BRIL Luminosity Anomaly Detection Algorithm

## Setting up the data and environment.
The code provided in the python notebooks in this repository are designed for individual anomaly selection of the both the PLT channels at CMS and BCM1F channels, and has been further extended to perform a more general channel selection of PLT and BCM1f data. The BCM1F detection model will be published soon and is designed the same way as the PLT detection model except to handle more channels. 

This code isn't ML based, so no training of a model is required. The python package adtk must be installed to ensure the code runs. In my jupyter notebook I have the first cell dedicated in the code to ensuring that adtk is installed and located in the correct python path, which is very useful if running on CERN servers and using the SWAN interface. Use python version 3.9.12 when running the code on this page. If you are running through a terminal, simply ensure adtk is intalled prior by running the following and then importing the following packages:

```
!pip install adtk
import pathlib
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import tables
from adtk.data import validate_series
from adtk.detector import ThresholdAD, LevelShiftAD
from datetime import datetime
import math
import warnings
import os
import scipy.optimize as opt  
from IPython.display import display_html
from IPython.core.display import display, HTML
from scipy.optimize import curve_fit
from scipy.optimize import least_squares
from sklearn.mixture import GaussianMixture
from sklearn.preprocessing import StandardScaler
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

## Using The Anomaly Selection Algorithms

There are a few ways to perform anomaly detection in this code. The first, marked by the heading cell ```Begin Channel Selection```, performs a more widespread anomaly detection model that checks to see if the channels have passed or have failed and uses the main function called ```channel_selection```. We take into account a few different conditions for a channel to be flagged as bad. First, we check the total number of final anomalies flagged, which varies based on how spread out the luminosity measurements are for a particular fill - ie, a fill that ranges over 20,000 luminosity when plotted will be treated differently than a fill that ranges over 1,000 luminosity. If enough anomalies are flagged, we then check the distribution of those anomalies to see how spread out or compact in a region they are detected. 

Two different situations are taken into account when finding anomalies: the ratio plot and the difference plot. When checking for anomalies in PLT, BCM1F, or HF data we need to ensure we don't accidentally flag the emittance scan as an anomaly - since by eye it looks like an anomaly that extends across all channels. Thus, we try to eliminate the emittance scan by taking the difference between the channel we're checking and the reference channel and the ratio between the selected channel and the reference channel. If a channel fails the channel selection, then we move on to using the ```full_automated``` function. This displays how the anomalies are distributed and highlights what threshold failed causing the channel to be marked as bad. From here the user can determine if the channel is actually okay and just barely failed or if some portion/all of the data needs to be removed.

In the code I have already selected parameters for the different adtk models. If needed these can be changed. Below explains each type of model I've used that together forms the anomaly detection algorithm:
* __threshold_ad__: _detects anomalies outside a set threshold range; for the sake of the model, we flag anomalies that lie +- some percentage outside the calculated mean value of the plot_ 
* __levelshift_ad__: _detects anomalies that are due to a massive spike in data_

The following are my current set parameters and my recommended ranges to remain inside:
* __percentage__ (threshold): _20%, stay between 15-25% depending how wide a range you want to cover_
* __levelshift__ (levelshift): _400, stay between 250-500 depending how strong of a spike detection you want (smaller number means more is detected)_
* __window__ (levelshift): _2, stay between 1-3 depending how strong of a spike detection you want (smaller number means more is detected)_


## Crafting Histograms

Before crafting histograms, run the ```process_fill``` function to append the data to csv files in order to study many of the fills at once.

It's important to visualize the distribution of data for the detectors. We have multiple histograms displayed throughout the code. The first histogram is easily recognized by the heading cell ```Display histograms for all fills studied to see how PLT data has shifted overtime```, which may take a minute or so to run depending on how many fills you have appended to the csv files. For this histogram, each fill is plotted as its own histogram so the data is not stacked. By comparison, the following histogram plots the stacked version with a gaussian mixing matrix fit - however this most likely won't work as PLT data tends to shift a lot and become un-gaussian in shape. 

We can also visualize a set number of fills closest in date-time to the fill just studied to determine if there are any immediate shifts in behavior in the detector channels that lend itself to removing that channel from the final data - thus catching information that wasn't caught by the channel selection tool. A table is then crafted to visualize the overall information about the channels and can give a ranking as to which channels are best and which are worst.

The ```bcm1f_anomaly_algorithm.ipynb``` is practically the same format just with slightly different variable names. Note in both notebooks there are guiding comments and header cells that match this README file. If there are questions or issues running the code, reach out to kmream@umich.edu.

Et voila! You've just performed channel and/or anomaly detection for PLT and BCM1F data! 
