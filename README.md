# Awesome Sensor Logger
This repository contains a collection of tools, resources and sample code to use alongside the Sensor Logger app.

- [The Sensor Logger App](#the-sensor-logger-app)
- [Getting Started with Data Analysis](#getting-started-with-data-analysis)
  * [Recommended Tools](#recommended-tools)
  * [Understanding Timestamps](#understanding-timestamps)
  * [File Handling](#file-handling)
  * [Accessing Recording Metadata](#accessing-recording-metadata)
  * [When to Use Uncalibrated Data](#when-to-use-uncalibrated-data)
  * [Plotting Data](#plotting-data)
  * [Diagnosing Sampling Rates](#diagnosing-sampling-rates)
  * [Mapping GPS Tracks](#mapping-gps-tracks)
  * [Aligning & Interpolating Measurements Across Sensors](#aligning---interpolating-measurements-across-sensors)
  * [Smoothing & Denoising](#smoothing---denoising)
  * [Fourier Transforms](#fourier-transforms)
  * [Track Simplification](#track-simplification)
  * [Peak Detection](#peak-detection)
  * [Activity Detection & Semantic Segmentation](#activity-detection---semantic-segmentation)
  * [Time Series Classification](#time-series-classification)
  * [Removing Duplicated Entries](#removing-duplicated-entries)
  * [Audio Analysis](#audio-analysis)
  * [Converting to GPX / InfluxDB](#converting-to-gpx---influxdb)
- [Live Data Streaming](#live-data-streaming)
- [Further Use Cases & Applications](#further-use-cases---applications)
- [Contribute](#contribute)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

<img width="970" alt="hero" src="https://user-images.githubusercontent.com/30114997/173469460-f20062ab-7b47-47bf-9f93-a266fa457ae9.png">

## The Sensor Logger App
Sensor Logger is a free, easy-to-use, cross-platform data logger that logs readings from common motion-related sensors on smartphones. Once completed, recordings can be exported as a zipped CSV file, JSON or be viewed within the app via interactive plots. The iOS version comes with a free companion watch app that allows you to log heart rate & wrist motion, and control recording sessions remotely from your wrist. Supported sensors and measurements include:

- Device Acceleration (Accelerometer; Raw & Calibrated)
- Gravity Vector (Accelerometer)
- Device Rotation Rate (Gyroscope; Raw & Calibrated)
- Device Orientation (Gyroscope)
- Magnetic Heading (Magnetometer; Raw & Calibrated)
- Barometric Altitude (Barometer)
- GPS Coordinate, Altitude, Speed & Heading
- Proximity Sensor (Android Only)
- Audio Recording (Microphone)
- Ambient Sound Level (Microphone)
- Heart Rate (via Companion Watch App)
- Wrist Motion (via Companion Watch App)
- Annotations (Timestamp and optional accompanying text comment)

Learn more and download Sensor Logger at www.tszheichoi.com/sensorlogger. 

| Android | iOS |
|:-:|:-:|
| [<img src="https://play.google.com/intl/en_us/badges/static/images/badges/en_badge_web_generic.png" height="50">](https://play.google.com/store/apps/details?id=com.kelvin.sensorapp&pcampaignid=pcampaignidMKT-Other-global-all-co-prtnr-py-PartBadge-Mar2515-1) | [<img src="https://developer.apple.com/app-store/marketing/guidelines/images/badge-example-preferred_2x.png" height="50">](https://apps.apple.com/app/id1531582925) |

## Getting Started with Data Analysis
Measurements made using the Sensor Logger can be exported in either `.csv` or `.json` formats. For data analysis, the former is recommended. See www.tszheichoi.com/sensorloggerhelp for more about how exporting works.

### Recommended Tools
Python is recommended for analysing the outputs from Sensor Logger. For interactive data exploration and visualisation, use Jupyter Notebooks, which you can run for free easily with tools like Google Colab or Deep Notes. Your typical data science Python packages apply:

- Pandas for CSV parsing and manipulation
- Numpy for data analysis
- SciPy for time series analysis
- Folium / Leaflet.js for geospatial visualisation
- Dash / Flask for data streaming

### Understanding Timestamps
All exported data have synchronised time stamps, meaning they can be cross-referenced. However, they do not necessarily align due to varied sampling rates. Note the followings:
- The `time` column is the UNIX epoch timestamp of the measurement as reported by the sensors in nanoseconds. You can use tools like https://www.epochconverter.com/ to convert them to readable timestamps. By definition, these are UTC times -- whereas the filenames are in local times. 
- The `seconds_elapsed` column is the number of seconds since you tapped the Start Recording button. Note that some entries could be negative, meaning the measurements were made *before* the start of the recording, but are reported by your phone *after* the tap due to buffering or caching. 
- Please note that the accuracy of timestamps relies on accurate system timestamps. Please make sure your phone’s time is accurate to ensure physically correct timestamps. If your phone changes time zone mid-recording, it may also lead to unpredictable behaviour. 

### File Handling
- Normally, just use https://docs.python.org/3/library/zipfile.html, but for batch unzipping in Python, read https://superfastpython.com/multithreaded-unzip-files/
- Reading `.csv`: Use `pandas`, see below.
- Reading audio file: Depends on whether you have compression enabled in the Sensor Logger's settings. 
    - Do read https://hackernoon.com/audio-handling-basics-how-to-process-audio-files-using-python-cli-jo283u3y
    - Use https://librosa.org/doc/latest/index.html

### Accessing Recording Metadata
The `metadata.csv` file contains information about the device that performed the logging.
- version: The schema version of the exported data. This is different to the version of the app. When this version increments, you may have to update your data analysis script as things such as the column names, file names, or data structure may have changed. 
- device name: The name of the device used for recording.
- recording time: The start time of the recording in UTC.
- platform: Either iOS or Android.

### When to Use Uncalibrated Data
Sensor Logger gives you the option to log raw, uncalibrated data from the accelerometer, gyroscope and magnetometer. Calibrated data is always logged. The raw stream is useful for performing lower-level post-processing or custom sensor fusion. If in doubt, *always* use the calibrated version unless you have a good reason not to. 

### Plotting Data
Use `pandas` to import `.csv` and convert timestamps. Use Plotly to visualise data interactively. Here, we show the acceleration experienced by a user during a roller coaster ride. 
```
import pandas as pd
import plotly.graph_objects as go

df = pd.read_csv('Accelerometer.csv')
df.index = pd.to_datetime(df['time'], unit = 'ns')

fig = go.Figure()

for axis in ['x', 'y', 'z']:
    fig.add_trace(go.Scatter(x = df.index, y = df[axis], name = axis))

fig.show()
```
<img width="937" alt="simple_accel" src="https://user-images.githubusercontent.com/30114997/173203712-3079afc2-a210-482a-9073-e4b2e31281b8.png">

Some sensors may also report uncertainties, which is important for analysis and multi-sensor fusion. For example, the speed from GPS has associated errors. 

```
import pandas as pd
import plotly.graph_objects as go

df = pd.read_csv('Location.csv')
df.index = pd.to_datetime(df['time'], unit='ns')

fig = go.Figure()

fig.add_trace(go.Scatter(x=df.index, y=df['speed'], mode='markers',
              error_y={'type': 'data', 'array': df['speedAccuracy']}))

fig.show()
```
<img width="879" alt="error_gps" src="https://user-images.githubusercontent.com/30114997/173204026-a55c2f9e-1f52-44d5-ba4e-6f3a69eeee72.png">

### Diagnosing Sampling Rates
Real-world measurements are never evenly sampled. It is important to double-check before any analysis. For example, this particular GPS location data recorded on an iPhone has `avg_sampling_rate` of 0.93Hz, but the measurement gap ranges between 0.99 to 12 seconds.
```
import pandas as pd
import numpy as np

df = pd.read_csv('Location.csv')
consecutive_deltas = df['seconds_elapsed'].diff()
avg_sampling_rate = 1 / np.mean(consecutive_deltas)
shortest_gap = np.min(consecutive_deltas)
maximum_gap = np.max(consecutive_deltas)
total_num_of_samples = len(df.index)
```
To resample your data, for example, to minute periods:
```
import pandas as pd

df = pd.read_csv('Location.csv')
df.index = pd.to_datetime(df['time'], unit = 'ns')
df.resample('1T').median()
```
Also read https://www.earthdatascience.org/courses/use-data-open-source-python/use-time-series-data-in-python/date-time-types-in-pandas-python/resample-time-series-data-pandas-python/

To understand the implications and robustness of resampling and dealing with missing data in motion-related time series, consult the following:

- https://towardsdatascience.com/preprocessing-iot-data-linear-resampling-dde750910531
- Impact of Reduced Sampling Rate on Accelerometer-based Physical Activity Monitoring and Machine Learning Activity Classification: https://www.medrxiv.org/content/10.1101/2020.10.22.20217927v1.full
- Sensor Data Augmentation by Resampling for
Contrastive Learning for Human Activity
Recognition: https://arxiv.org/pdf/2109.02054.pdf

### Mapping GPS Tracks
Use tools like Folium, which is built on top of leaflet.js to overlap GPS tracks on a map
- https://towardsdatascience.com/build-interactive-gps-activity-maps-from-gpx-files-using-folium-cf9eebba1fe7
- https://towardsdatascience.com/simple-gps-data-visualization-using-python-and-open-street-maps-50f992e9b676

```
import folium
import pandas as pd

df = pd.read_csv("Location.csv")
coords = [(row.latitude, row.longitude) for _, row in df.iterrows()]

my_map = folium.Map(location=[df.latitude.mean(), df.longitude.mean()], zoom_start=16)
folium.PolyLine(coords, color="blue", weight=5.0).add_to(my_map)
```

<img width="972" alt="thorpe_park_track" src="https://user-images.githubusercontent.com/30114997/173207512-cffd38f0-400f-44ef-b3b3-2e7963a51a88.png">


Alternatively, convert your exported data to GPS using https://github.com/mhaberler/sensorlogger-util, and then upload to Google Maps for visualisation, following, for example, https://www.alphr.com/gpx-google-maps/. 

### Aligning & Interpolating Measurements Across Sensors
Often, one has to align measurements across sensors -- for instance, gyroscope and accelerometer so that you can apply rotation matrixes to the acceleration vectors. 
- Option 1: Perform an outer join and then interpolate missing values for both sensors. By default, `pandas` interpolates linearly, but see the documentation for more advanced options.
```
import pandas as pd
import numpy as np

df_gyro = pd.read_csv('Gyroscope.csv')
df_acce = pd.read_csv('Accelerometer.csv')

for df in [df_gyro, df_acce]:
    df.index = pd.to_datetime(df['time'], unit = 'ns')

df = df_gyro.join(df_acce, lsuffix = '_gyro', rsuffix = '_acce', how = 'outer').interpolate()
```
- Option 2: Interpolate one sensor onto the timestamps of another -- probably better if the two sensors have wildly different sampling rates. For example: 
```
np.interp(df_acce.index, df_gyro.index, df_gyro['x'])
```
Using option 1 above, the altitude readings from both the GPS and barometer are resampled and aligned. As you can see, the GPS is not great at characterising rapid altitude changes, and exhibits biases during transient periods.

<img width="929" alt="resampled_altitudes" src="https://user-images.githubusercontent.com/30114997/173237727-d48fa023-4d47-48db-91b0-48e15fb060e1.png">

For more complex alignment needs -- such as aligning with measurements from sources other than Sensor Logger, you may need techniques such as cross-correlation or dynamic time warping:
- https://towardsdatascience.com/dynamic-time-warping-3933f25fcdd
- https://dtaidistance.readthedocs.io/en/latest/usage/dtw.html
- https://dynamictimewarping.github.io/

### Smoothing & Denoising
Different applications require different smoothing and denoising strategies. This scipy cookbook has some handy code you can borrow:
- https://scipy-cookbook.readthedocs.io/items/SignalSmooth.html
- https://scipy-cookbook.readthedocs.io/items/SavitzkyGolay.html
- Also try `scipy.ndimage.median_filter`

Referencing the previous section on interpolation, performing downsampling with a suitable aggregator (e.g. median) or interpolation with suitable spline or polynomial functions may also achieve smoothing. 
### Fourier Transforms
Use Fourier transforms to understand any periodicity in your logged values. For example, you can detect walking by simply thresholding the gravity vector measurements in frequency space. Here is a simple example of Fourier transforming the gravity measurements taken on [Zodiac, a spinning ride at Thorpe Park](https://en.wikipedia.org/wiki/Zodiac_(ride)). As you can see, there is a peak around 0.25 Hz (i.e. the ride goes around one revolution every 4 seconds). 

```
import pandas as pd
import numpy as np

df = pd.read_csv('Gravity.csv')
ts = pd.to_datetime(df['time'], unit = 'ns')
period = np.nanmedian(ts.diff().dt.total_seconds())

sp = np.fft.fft(df['x'])
freq = np.fft.fftfreq(len(df.index), period)

mask = (freq >= 0) &(freq < 1)

fig = go.Figure()
fig.add_trace(go.Scatter(x = freq[mask], y = [np.linalg.norm(s) for s in sp[mask]]))
fig.show()
```

<img width="935" alt="zodiac" src="https://user-images.githubusercontent.com/30114997/173238828-438cf3db-e767-4dd3-a0af-7295393aae4e.png">

### Track Simplification
You can use algorithms like Douglas-Peucker to simplify recorded GPS tracks to save storage space for long recordings. A Sensor Logger specific implementation can be found here: https://github.com/mhaberler/sensorlogger-util/blob/master/simplify.py

### Peak Detection
- https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.find_peaks.html
- https://en.wikipedia.org/wiki/Topographic_prominence
- https://stackoverflow.com/questions/1713335/peak-finding-algorithm-for-python-scipy

### Activity Detection & Semantic Segmentation
- A Comprehensive Study of Activity Recognition Using Accelerometers: https://www.researchgate.net/publication/325470495_A_Comprehensive_Study_of_Activity_Recognition_Using_Accelerometers
- Useful tool for time series data mining tasks: https://stumpy.readthedocs.io/en/latest/
- Kang et al. 2017 A Novel Walking Detection and Step Counting Algorithm Using Unconstrained Smartphones

### Time Series Classification
- https://towardsdatascience.com/a-brief-introduction-to-time-series-classification-algorithms-7b4284d31b97

### Removing Duplicated Entries
Sometimes, erroneous sensors may report the same value twice with identical timestamps, likely due to caching. Try something like this to remove them:
```
def remove_duplicated_rows(df: pd.DataFrame):
    _df = df[
        ~(df["timeStamp"].diff() < 0)
    ]  # be careful that diff() gives nan for first row
    if len(_df.index) != len(df.index):
        print(
            f"duplicated rows detected in the input file ({len(df.index) - len(_df.index)}"
        )
    return _df
``` 
### Audio Analysis
`ffmpeg-python` (https://github.com/kkroening/ffmpeg-python) has some valuable examples of interesting audio analysis:
- Splitting silence: https://github.com/kkroening/ffmpeg-python/blob/master/examples/split_silence.py
- Audio transcription with Google Cloud: https://github.com/kkroening/ffmpeg-python/blob/master/examples/transcribe.py

`pyAudioAnalysis` is also worth checking out for audio feature extraction, classification and segmentation: https://github.com/tyiannak/pyAudioAnalysis

### Converting to GPX / InfluxDB
Michael Haberler has helpfully put together a command-line tool `sensorlogger-utils` to take the JSON exported from Sensor Logger and convert it to GPX or InfluxDB formats: https://github.com/mhaberler/sensorlogger-util

```
python sensorlogger -g <json file>
```
```
python sensorlogger.py -2 [--bucket sensorlogger] --token xxx  --org yyyy --url http://host:8086 2022-06-14_03-15-05.json
```

## Live Data Streaming

As of version 1.10, Sensor Logger supports pushing live data via HTTP. This can be enabled by tapping the gear icon on the Logger page. All enabled sensors during a recording will be streamed every 200ms to the specified URL. To display the streamed data, you will need to set up a websever on another computer. 

<img width="586" alt="Screenshot 2022-07-12 at 09 33 42" src="https://user-images.githubusercontent.com/30114997/178447207-69379436-ca6e-4836-b0b8-8bac3b120ed0.png">

The schema of the streamed data is a json string of format `{messageId: int, payload: List}`. The `payload` is a list of `{time: int, name: str, values: Dict}`, where the name is the name of the sensor. The time is in UTC epoch nanoseconds. The `messageId` is useful because the messages can be recieved out-of-order, which may need to be handled depending on your use case.

To simply consume and explore the data, you may want to use something like https://requestbin.com/. To plot the data in real-time, you may need something more custom. See https://github.com/mhaberler/sensorlogger-telegraf for a solution using telegraf. 

Here is an example Python implementation using Plotly Dash to get you started. Dash is powered by Flask under the hood, and provides an easy way to set up a web server for real-time, interactive data visualisation. This code listens on the `/data` endpoint, filters only the values from the accelerometer and plots it. The `update_graph()` callback is triggered every `UPDATE_FREQ_MS`, and update the plot with any accumulated measurements so far. You will have to custimse this script yourself if you want to plot measurements from other sensors. 

```
import dash
from dash.dependencies import Output, Input
from dash import dcc, html, dcc
from datetime import datetime
import json
import plotly.graph_objs as go
from collections import deque
from flask import Flask, request

server = Flask(__name__)
app = dash.Dash(__name__, server=server)

MAX_DATA_POINTS = 1000
UPDATE_FREQ_MS = 100

time = deque(maxlen=MAX_DATA_POINTS)
accel_x = deque(maxlen=MAX_DATA_POINTS)
accel_y = deque(maxlen=MAX_DATA_POINTS)
accel_z = deque(maxlen=MAX_DATA_POINTS)

app.layout = html.Div(
	[
		dcc.Markdown(
			children="""
			# Live Sensor Readings
			Streamed from Sensor Logger: tszheichoi.com/sensorlogger
		"""
		),
		dcc.Graph(id="live_graph"),
		dcc.Interval(id="counter", interval=UPDATE_FREQ_MS),
	]
)


@app.callback(Output("live_graph", "figure"), Input("counter", "n_intervals"))
def update_graph(_counter):
	data = [
		go.Scatter(x=list(time), y=list(d), name=name)
		for d, name in zip([accel_x, accel_y, accel_z], ["X", "Y", "Z"])
	]

	graph = {
		"data": data,
		"layout": go.Layout(
			{
				"xaxis": {"type": "date"},
				"yaxis": {"title": "Acceleration ms<sup>-2</sup>"},
			}
		),
	}
	if (
		len(time) > 0
	):  #  cannot adjust plot ranges until there is at least one data point
		graph["layout"]["xaxis"]["range"] = [min(time), max(time)]
		graph["layout"]["yaxis"]["range"] = [
			min(accel_x + accel_y + accel_z),
			max(accel_x + accel_y + accel_z),
		]

	return graph


@server.route("/data", methods=["POST"])
def data():  # listens to the data streamed from the sensor logger
	if str(request.method) == "POST":
		print(f'received data: {request.data}')
		data = json.loads(request.data)
		for d in data['payload']:
			if (
				d.get("name", None) == "accelerometer"
			):  #  modify to access different sensors
				ts = datetime.fromtimestamp(d["time"] / 1000000000)
				if len(time) == 0 or ts > time[-1]:
					time.append(ts)
					# modify the following based on which sensor is accessed, log the raw json for guidance
					accel_x.append(d["values"]["x"])
					accel_y.append(d["values"]["y"])
					accel_z.append(d["values"]["z"])
	return "success"


if __name__ == "__main__":
	app.run_server(port=8000, host="0.0.0.0")
```

Run this Python script and visit `http://localhost:8000/` on your computer. Then you have to enter the correct Push URL in Sensor Logger on your phone under the settings page. To find out the localhost of the device you are running the websever on, you can, for example, do something like this in Python. 

```
import socket
hostname = socket.gethostname()
print(socket.gethostbyname(hostname))
```

For example, if it returns `192.168.1.168`, then you want to enter `http://192.168.1.168:8000/data` in Sensor Logger. Use the "Tap to Test Pushing" button to test whether Sensor Logger can properly reach the endpoint. If you get a 200 response, then you are good to go! Start a recording as usual, and you should begin to see data being streamed in:

<img width="1440" alt="Screenshot 2022-07-10 at 10 23 03" src="https://user-images.githubusercontent.com/30114997/178138947-8522be59-4c0b-4966-84ad-49416fda5fc0.png">


## Further Use Cases & Applications
Based on [user-submitted feedback](https://www.tszheichoi.com/sensor-logger-feedback), Sensor Logger is being use for a lot of applications -- for researchers and hobbyists alike. Here are a few to get you started. Let me know, and I will feature your use case here as well!
- Hotair balloon tracking
- Flight data logging
- Wearable technology research
- Pedestrian dead reckoning
    - https://github.com/IdoMatan/PDR
- Roller coaster tracking
    - https://www.tszheichoi.com/coaster
- Sports research
    - https://www.tandfonline.com/doi/abs/10.1080/02640414.2021.1993640?journalCode=rjsp20
- Vehicle tracking
- Figure skating analysis
- Whale watching
- Video stabilisation
    - https://docs.gyroflow.xyz/
- Inertial navigation
- Sensor fusion / Kalman filtering / Data assimilation
    - https://scipy-cookbook.readthedocs.io/items/KalmanFiltering.html
    - https://www.youtube.com/watch?v=TmBEryh2OXY
- GPS data logging for photo / video geolocation. 

## Contribute
Please submit a PR if you have scripts or links that may be useful for other users. I will also feature any project that uses Sensor Logger, integrated as part of a larger workflow. 
