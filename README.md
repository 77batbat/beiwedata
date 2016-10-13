# Introduction
`beiwedata` is a set of Python scripts designed to help munge, analyze, and manipulate data generated by the Beiwe application. 

# Table of Contents
- [Usage](#usage)
- [Data overview](#data-overview)
    - [Timestamps](#timestamps)
    - [Accelerometer](#accelerometer-data)
    - [Bluetooth](#bluetooth)
    - [Phone calls](#phone-calls)
    - [GPS](#gps)
    - [IDs](#ids)
    - [Application log files](#beiwe-logs)
    - [Power state](#power-state)
    - [Survey answers](#survey-answers)
    - [Survey timings](#survey-timings)
    - [Text messages](#text-messages)
    - [Voice memos](#voice-memos)
    - [WiFi logs](#wifi)
- [Known data issues](#known-data-issues)
- [Functions](#functions)
- [Example plots](#example-plots)



# Usage
`from beiwedata import *` is the standard way to import and this document will assume `beiwedata` has been imported to the top level. If you `import as`, please adjust the usage examples accordingly.

# Data overview
There are a total of 12 types of files generated by the `beiwe` app. Files are stored as comma-separated values (`csv`) files and **every** file contains column headers (including empty files). Files are created periodically by the app, and empty files occur when the application records no data during that period. For example, the device runs its periodic "WiFi" check, but the WiFi transceiver may be disabled or it may simply pick up no local WiFi networks. This file is later "retired" by the app so that it may be uploaded and deleted from the device.

\#|Data stream|File prefix|\# Col
:-:|-----------|-----------|:---:
1|Accelerometer|`accel`|5
2|Bluetooth|`bluetoothLog`|3
3|Phone calls|`callLog`|4
4|GPS|`gps`|5
5|IDs|`identifiers`|4
6|Beiwe logs|`logFile`|1
7|Power State|`powerState`|2
8|Survey (1)|`surveyAnswers`|5
9|Survey (2)|`surveyTimings`|6
10|Text messages|`textsLog`|5
11|Voice memos|`voiceRecording`|NA
12|WiFi|`wifiLog`|3

## Timestamps
Each file prefix above is followed by `_[timestamp].csv`. The timestamp is Java time in UTC, which is milliseconds from epoch, and represents the *time of file creation*. Thus, note that this timestamp may differ from the timestamp you see online which represents the time of file upload. (**NOTE:** The app is designed to only upload when it is connected to WiFi.)

In order to use most conversion tools (which are designed for [Unix time](https://en.wikipedia.org/wiki/Unix_time) — i.e.,  seconds from epoch), simply perform integer division by 1000. For example, in `Python`, one must first perform `int(timestamp / 1000)` before using the `datetime` module to convert to human-readable time. In `Microsoft Excel` one might use a formula such as `=((timestamp/1000) / 86400) + 25569` and then convert the cell to datetime.

See [EpochConverter](http://www.epochconverter.com/) for more ways to manipulate Unix time into various programming languages.

## Accelerometer data
Accelerometer files contain 5 columns: `timestamp`, `accuracy`, `x`, `y`, and `z`. Accelerometer `on` and `off` periods may vary by study, and sampling rates rates vary by device and possibly by Android OS version as well. Many devices limit sampling while the screen is off, though a few disable the accelerometer completely. High frequency sampling occurs when the screen is on, but that rate differs by OS and phone.

**NOTE:** The bounds of the `x`, `y`, and `z` values are specific to each phone model. In all our test data the bounds are [-20, 20], but research by the developers indicates that for some devices it is [-10, 10]. It is unclear if that bound value is derived after accounting for acceleration due to gravity.  The data recorded by the app is raw accelerometer data and has _not_ been modified to remove acceleration due to gravity.

## Bluetooth
Bluetooth files contain: `timestamp`, `MAC`, and `RSSI`. Note that `MAC` is actually the *hashed* MAC address of other devices. `RSSI` is in dBm.
Note: due to restrictions starting in Android version 6 only a few devices can now report their mac address, instead these should report either "N/A" for their Mac address.

## Phone calls
Call logs contain: `hashed phone number`, `call type`, `date`, and `duration in seconds`. Note that `date` is the equivalent of `timestamp` in other files (i.e., it is not a human-readable datetime object).

## GPS
GPS files contain: `time`, `latitude`, `longitude`, `altitude`, and `accuracy`. Note that `time` here is the equivalent of `timestamp` in other places. Again, programmers may change this in future versions to be more consistent. Accuracy is the accuracy in meters.

**NOTE:** Altitude accuracy varies _significantly_ from device to device, see [this StackOverflow post](http://stackoverflow.com/questions/9361870/android-how-to-get-accurate-altitude) for details. The value of the `accuracy` field only applies to horizontal accuracy.

## IDs
ID files should only contain one row and in most cases will only have one instance; the file will get recreated if a user-id is re-registered, and it will contain different identifying information if the user is re-registered on a different device. This file will contain the users own `patient id`, `(hashed) MAC`, `(hashed) phone_number`, and `device_id`.

**NOTE:** The device ID is sourced from an operating system value called ANDROID\_ID. It is unique to the device, but will change if a user does a factory reset on their device and then reinstalls the Beiwe app. The hashed MAC is of the phone's _BlueTooth_ MAC address.

## Beiwe logs
Beiwe log files are app-generated messages and should be used only for diagnostic purposes. Data recorded in this log file are subject to change between any version of the Beiwe app, and conform to no consistent formatting.

## Power State
Power logs have two columns: `time` and `event`. Note that `time` here is the equivalent of `timestamp` elsewhere. Power state events are things like `screen on`, `screen off`, `power connected` and `power disconnected`.

The following data points are introduced in Beiwe version 10.
Note: the strings are intentionally over-explicit because Android only provides a notification that state has changed, and the app then has to check the current state. These operation non-atomic, and so the check has a tiny potential to be out of date.

1) Doze:
Doze is a new power state (technically a sleep state) added in Android 6. You can view some [useful documentation here](https://developer.android.com/training/monitoring-device-state/doze-standby.html#understand_doze).
Summary: the Doze state is entered only if a device is unplugged, the screen is off, and the accelerometer indicates no or minimal device motion. It leaves this state _periodically_ to run scheduled tasks (a "maintenance window"), but does so with increasing-in-length sleep periods. The device also wakes up if any of the previously mentioned requirements are interrupted, or if the operating system identifies a "significant location change." _All scheduled data recording sessions and survey notifications are delayed until the device wakes up or enters a maintenance window.

`Device Idle (Doze) state change signal received; device in idle state.`
`Device Idle (Doze) state change signal received; device not in idle state.`

2) "Power Save State"
This was introduced in Android 5, the extent of documentation is "When in this mode, applications should reduce their functionality in order to conserve battery as much as possible." Apparently actually doing something, like ... reducing GPS frequency?, is up to the manufacturer, but may become one of those things that Android enforces globally in the future. It is not well defined when precisely this mode is entered, it is not well defined what this mode actually does, and currently (Android 6) we do not believe it has a determinable effect on Beiwe Android data collection.

`Power Save Mode state change signal received; device in power save state.`
`Power Save Mode change signal received; device not in power save state.`


## Survey Answers
Variable depending on the survey. (note that this file also contains the text of any questions)

## Survey Timings
Variable depending on the survey.

## Text messages
Text logs contain: `timestamp`, `hashed phone number`, `sent vs received`, `message length`, and `time sent`. In general, `time sent` should be ignored. It is theoretically the time the message was sent (from somebody else) while `timestamp` is the time that message would have been received by the user. In practice, these should be identical or very similar.

## Voice memos
Self-recorded voice memos.  Will vary by study.  Voice recordings are in one of two formats: mp4 and wav files.  Mp4 files use AAC compression, wav files contain PCM data.  All audio files are a single channel.

## WiFi
Contains `hashed MAC`, `frequency`, and `RSSI`. Frequency will always be 2.4GHz or 5GHz so I'm not sure how useful that is. `RSSI` is in dBm.

# Known Data Issues
- The `textsLog` data often contain duplicates (usually 3 or 4 identical rows). This seems to be more common for sent texts but can occur for received texts. 
- Sometimes, two accelerometer data points will have identical timestamps. This is caused by the accelerometer reporting and the app processing two measurements within the same millisecond of time, which is below the resolution of the app to record. Relatively rare (~ `1300 / 1.4 million` in my data).
- Users will sometimes receive a `Bluetooth Share has stopped` error message, at which point Bluetooth recording will probably fail. This is a problem with Android OS 4.3 to 4.4.3, and is caused by the device receiving too many Bluetooth LE signals too quickly. This is not a problem with the Beiwe app, it is a bug in Android's networking software stack, and there is nothing we can do about it.
- All devices receive Bluetooth beacons correctly, but devices running versions of Android before Lollipop (Android OS 5) will not always transmit Bluetooth beacons. This will bias the unique devices count (downwards).
- Android version 6 significantly restricts access to the Bluetooth Mac address, though some devices will still be able to report.
- Android version 6 implements a new power-saving behavior, but it is unknown which devices implement this feature _as documented_.  Documented behavior: if the device is stationary for some "long" period of time _and_ unplugged it will enter a low power mode in which it delays timer triggers.  The Beiwe Android App tries to handle these events gracefully, delaying data recording sessions until the device is picked up or otherwise interacted with. The inactivity time period is unknown; the determination of whether the device has been stationary is made by the Android OS monitoring data from the accelerometer, but there are indications that not all devices actually do this. Devices in Doze mode will periodically "wake up" to execute delayed timer events, but successive "wake up" events will occur less and less frequently, resetting when the device is picked up or otherwise interacted with.

# Functions
More information about each function can be found in the function's docstring (i.e., `help([function])` or `?[function]`). This is just a list of functions to give you an idea of what has already been done.

- `row_count()` -- Iterates through a `csv` file and finds the number of observations
- `list_data_files()` -- Iterates through a user directory and looks for `csv` files that match a certain datastream (and time frame). For example, will find just accelerometer files or only survey files. (Note that this will **not** find voice recordings.)
- `list_audio_files()` -- Iterates through a user directory and looks for audio files (can filter out by time frame).
- `convert_mp4()` -- Audio files are saved as `mp4` in order to save space, but Python can only analyze `wav` files. Thus, this is a wrapped for `FFmpeg` to convert `mp4` files to `wav` files. **NOTE:** You must install `FFmpeg`.
- `return_file()` -- Takes a `csv` and returns a list of lists (i.e., a list containing sublists which are rows). The first item in the list is a list of header names.
- `make_timestamp()` -- A convenience wrapper function for specifying a date/time and returning a timestamp in UTC (in either Unix or Java time).
- `import_df()` -- Takes a list of files (created by `list_data_files()`) and turns them into a single `pandas` DataFrame. 
- `ts_to_local()` / `ts_to_utc()` -- Just convenience functions that will quickly turn a timestamp into local time or UTC in human-readable format.
- `plot_accel()` -- Takes an accelerometer dataframe (generated by `import_df()`) and returns a plot of the specified time interval.
- `plot_wav()` -- Takes a `wav` file (converted from `convert_mp4()` and plots the amplitude by time.
- `describe_user()` -- Looks through a user directory and provides quick summary information about that user's data. For example, number of files (by data stream) or number of observations, or empty files.
- `plot_gps()` -- Plots a GPS dataframe (generated by `import_df()`). Must have `basemap` installed.
- `duplicates()` -- Loops through a text message dataframe (generated by `import_df()`) and finds duplicated according to a "sliding window" rule. Use with caution. This is my solution, but is probably not good enough for people interested in truly understanding the communication network of these users.
- `rank_mac()` -- Takes a WiFi or Bluetooth dataframe and finds the top `n` MAC addresses for a specified aggregation level (default is 15 seconds).
- `plot_most_macs()` -- Takes a `rank_mac()`-generated dataframe and plots the instances of observations for each MAC throughout the specified timeline.
- `plot_n_macs()` -- Takes a WiFi or Bluetooth dataframe and plots the number of unique MAC addresses for a specified time aggregation level and also plots the cumulative distribution of MAC addresses.
- `plot_calls_texts()` -- Takes both a call and a text dataframe and roughly plots them.

# Example plots
All example code below assumes you've run the following import code:

```
import matplotlib
import matplotlib.pyplot as plt
import pandas as pd
from beiwedata import *
```

## Voice recordings
![Voice plot](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/voice_plot.png)

This plot was generated using `plot_wav()`.

```
## Get audio files
audio_files = list_audio_files(fpath='./user1', mp4only=True)

## Convert one to .wav
convert_mp4(audio_files[-1])

## Plot it
plot_wav(audio_files[-1][:-4] + '.wav', psave=True)
```

## Accelerometer plots
![Accel plot](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/accel_plot.png)

Plot was generated using `plot_accel()`.

```
acc_files = list_data_files(fpath='./user2', stream='accel')
df_accel = import_df(acc_files)

start = make_timestamp(2015, 5, 5, 23, 58)
end = make_timestamp(2015, 5, 7, 0, 2)

plot_accel(df_accel, start, end, psave=True)
```

## GPS plots
![GPS plot](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/gps_plot.png)

```
fgps = list_data_files(fpath='./user3', stream='gps')
dgps = import_df(fgps, tstamp='time')
dgps = dgps[dgps.accuracy < 75]

start = make_timestamp(2015, 5, 7, 14, 28)
end = make_timestamp(2015, 5, 7, 14, 33)

## This adds the roads to my map. Remove the shpfile parameter
## if you are actually running this code
fig = plot_gps(dgps, start, end,
               shpfile='./mydata/converted_MA_roads/converted_MA_roads',
               shpname='roads',
               sspacer=[.0002, .00065], dscale=True, slength=.2)
plt.show()
fig.set_size_inches(6, 6)
fig.savefig('gps_plot.pdf', bbox_inches='tight')
plt.close('all')
```

## Calls and texts plot
![Calls and texts](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/call_text_figure.png)

```
cfiles = list_data_files(stream='call', fpath='user')
tfiles = list_data_files(stream='text', fpath='user')
tdf = import_df(tfiles)
cdf = import_df(cfiles)

fig, axes = plot_calls_texts(cdf, tdf, start_ts, end_ts)

axes.set_xlim([datetime.datetime(2015, 6, 22, 13),
               datetime.datetime(2015, 6, 22, 21)])

for label in axes.xaxis.get_ticklabels()[::2]:
    label.set_visible(False)

fig = plt.gcf()
fig.set_size_inches(12, 4)
fig.savefig('calls_and_texts.pdf', bbox_inches='tight')
```

## Bluetooth plots
![Bluetooth plots 1](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/bluetooth_top_devices.png)

```
bfiles = list_data_files(stream='blue', fpath='./mkdata')
bdf = import_df(bfiles)

start_ts = make_timestamp(2015, 5, 5, 23, 55)
end_ts = make_timestamp(2015, 5, 7, 0, 5)

freq_bt = rank_mac(bdf, start_ts, end_ts, n=10)
fig, axes = plot_most_macs(macdf = freq_bt)

axes.set_xlim([datetime.datetime(2015, 5, 5, 23, 55),
                datetime.datetime(2015, 5, 7, 0, 5)])
for label in axes.xaxis.get_ticklabels()[::2]:
    label.set_visible(False)
axes.set_title('Top Bluetooth Devices')
axes.set_ylabel('Unique Bluetooth Devices')
fig = plt.gcf()
fig.set_size_inches(12, 4)
fig.savefig('bluetooth_top_devices.png', bbox_inches='tight', dpi=150)
```
**NOTE:** See WiFi plots for variations of `plot_most_macs()`.

![Bluetooth plots 2](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/bluetooth_devices.png)

```
fig, axes = plot_n_macs(bdf, start_ts, end_ts)
axes[0].set_title('Bluetooth devices throughout the day')
for label in axes[1].xaxis.get_ticklabels()[1::2]:
    label.set_visible(False)
fig = plt.gcf()
fig.set_size_inches(12, 8)
fig.savefig('bluetooth_devices.png', bbox_inches='tight', dpi=150)
```

## WiFi plots
![Wifi plots 1](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_top_networks.png)

```
wfiles = list_data_files(stream='wifi', fpath='./mkdata')
wdf = import_df(wfiles)
start_ts = make_timestamp(2015, 5, 5, 23, 55)
end_ts = make_timestamp(2015, 5, 7, 0, 5)

freq_w = rank_mac(wdf, start_ts, end_ts, cname='hashed MAC', n=100)
fig, axes = plot_most_macs(macdf = freq_w, m='.')
axes.grid(False)
axes.set_title('Top WiFi Networks')
axes.set_ylabel('Unique WiFi Networks')
axes.set_ylim([0, 101])
for i, label in enumerate(axes.yaxis.get_ticklabels()):
    if (i % 10 != 0) and (i < len(axes.yaxis.get_ticklabels()) - 1):
        label.set_visible(False)
fig = plt.gcf()
fig.set_size_inches(18, 9)
fig.savefig('wifi_networks.png', bbox_inches='tight', dpi=150)
```

### With `strength` option

You may be interested in incorporating signal strength (`RSSI`) into these plots. Signal strength is highly variable by a number of external factors and it probably not something you want to plot the raw data for; however, `plot_most_macs()` does allow you to quickly plot percentiles of strength.

For example, assume we are the quartiles of signal strength, we just add `strength=4` to `plot_most_macs()`:
![Wifi plot with strength 4](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_top_networks_strength4.png)

```
fig, axes = plot_most_macs(macdf = freq_w, m='.', strength=4)
axes.grid(False)
axes.set_title('Top WiFi Networks with Signal Strength')
axes.set_ylabel('Unique WiFi Networks')
axes.set_ylim([0, 101])
for i, label in enumerate(axes.yaxis.get_ticklabels()):
    if (i % 10 != 0) and (i < len(axes.yaxis.get_ticklabels()) - 1):
        label.set_visible(False)
fig = plt.gcf()
fig.set_size_inches(18, 9)
fig.savefig('wifi_top_networks_strength4.png', bbox_inches='tight', dpi=150)
```

This colors by quantiles so any arbitrary (integer) number is allowed. However, the colormapping has been cut off from [0, 1] to [.3, 1] since 0 (i.e., white) comes out looking too faint. Thus, for quantiles larger than about 5 or 6, differences become hard to see. Here is one with `strength=25`: 

![Wifi plot with strength 25](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_top_networks_strength25.png)

### Wifi with `plot_others` option
We can also plot all others (that is, all devices not in ranking) by setting `plot_others=True`:

![Wifi plot with others](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_top_networks_others.png)

```
fig, axes = _macs(macdf = freq_w, m='.', plot_others=True)
axes.grid(False)
axes.set_title('Top WiFi Networks')
axes.set_ylabel('Unique WiFi Networks')
axes.set_ylim([-8, 101])
```
This should be considered a diagnostic convenience -- an easy way to tell if we are still collecting WiFi/Bluetooth data. However, note that this information is redundant with `plot_n_macs()`.

### Wifi with `plot_others` and `strength`
And obviously you can combine both `plot_others=True` with `strength` together:

![Wifi plot with strength and others](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_top_networks_strength4_others.png)

```
fig, axes = plot_most_macs(macdf = freq_w, m='.', strength=4,
                           plot_others=True)
axes.grid(False)
axes.set_title('Top WiFi Networks')
axes.set_ylabel('Unique WiFi Networks')
fig = plt.gcf()
fig.set_size_inches(18, 9)
fig.savefig('wifi_top_networks_strength4_others.png', bbox_inches='tight',
            dpi=150)
```


![Wifi plots 2](https://github.com/onnela-lab/beiwedata/blob/master/example%20plots/wifi_networks.png)

```
fig, axes = plot_n_macs(wdf, start_ts, end_ts, cname='hashed MAC')
axes[0].set_title('Wifi networks throughout the day')
fig = plt.gcf()
fig.set_size_inches(12, 8)
fig.savefig('wifi_networks.png', bbox_inches='tight', dpi=150)
```
