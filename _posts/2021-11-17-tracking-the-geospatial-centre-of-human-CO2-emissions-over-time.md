---
layout: single
title: Tracking the Geospatial Centre of Human-Induced CO<sub>2</sub> Emissions Over Time
categories: [Data analysis, Data visualisation, Climate, Python]
toc: true
toc_sticky: true
date: 2021-11-17
---

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    TeX: {
      equationNumbers: { autoNumber: "AMS" },
      tagSide: "right"
    },
    tex2jax: {
      inlineMath: [ ['$','$'], ["\\(","\\)"] ],
      displayMath: [ ['$$','$$'], ["\\[","\\]"] ],
      processEscapes: true
    }
  });
  MathJax.Hub.Register.StartupHook("TeX AMSmath Ready", function () {
    MathJax.InputJax.TeX.Stack.Item.AMSarray.Augment({
      clearTag() {
        if (!this.global.notags) {
          this.super(arguments).clearTag.call(this);
        }
      }
    });
  });
</script>
<script type="text/javascript" charset="utf-8"
  src="https://cdn.jsdelivr.net/npm/mathjax@2/MathJax.js?config=TeX-AMS_CHTML">
</script>

<p>
    The recent <a href="https://ukcop26.org/">COP26</a> climate change
    conference has been at the forefront of news reporting and social media
    discussions for several weeks. The <a
    href="https://github.com/tjukanovt/30DayMapChallenge">30 Day Map
    Challenge</a> has also been prevalent in my Twitter feed this month and,
    although I don't have time to take part in the challenge itself (writing a
    PhD thesis is quite time consuming!), I thought it timely to follow up on a
    mini-project idea I've had for a while &#8211; visualising the change in the
    weighted average position of human-induced CO<sub>2</sub> emissions in time.
</p>

<p>
    I used Python 3 for all of the data manipulation and visualisation. The code
    is available <a href="https://github.com/georgeholt1/geospatial_CO2">on my
    GitHub</a>.
</p>

<h2 id="The data">The data</h2>

<p>
    Two sources of data are required:
    <ol>
        <li>A coordinate reference for each country,</li>
        <li>A time series of per country CO<sub>2</sub> emissions.</li>
    </ol>
</p>

<h3 id="Country coordinates">Country coordinates</h3>

<p>
    The blog 'At These Coordinates' has a <a
    href="https://atcoordinates.info/resources/">resources page</a> with a
    treasure trove of useful data and tutorials. For this project, I grabbed a
    copy of the <a
    href="https://atcoordinates.files.wordpress.com/2018/01/country_centroids.zip">country
    centroids</a> data set, which contains latitude and longitude coordinates
    for the geographic centre of each country in the world as of February 2012.
</p>

<p>
    Country borders can (and do) change over time, but assuming that they are
    static should be sufficient for this analysis.
</p>

<p>
    The other key simplification is that each country is assumed to emit all of
    its CO<sub>2</sub> homogeneously across its area, which is equivalent to
    assuming that it is all emitted from the geographic centre. While not
    perfect, particularly for countries with highly localised industrial
    regions, this assumption is reasonable as the intra-country length scales
    are small relative to the global area.
</p>

<p>
    Loading the data is super simple using <a
    href="https://pandas.pydata.org/">pandas</a>:
</p>

```python
import pandas as pd
df_coords = pd.read_csv('./data/country_centroids.csv', sep='\t')
```

<h3 id="CO2 emissions data">CO<sub>2</sub> emissions data</h3>

<p>
    <a href="https://ourworldindata.org/">Our World in Data</a> is an excellent
    project with the stated aim of using "research and data to make progress
    against the world's largest problems". The website hosts a huge number of
    articles and visualisations in a huge range of topics including poverty and
    global inequality, technological innovation, health, and many more. The
    article on <a
    href="https://ourworldindata.org/co2-and-other-greenhouse-gas-emissions">CO<small>2</small>
    and Greenhouse Gas Emissions</a> has some wonderful insights and makes all
    of the data available via a <a
    href="https://github.com/owid/co2-data">repository on GitHub</a>. I have
    used a subset of data for this project; namely, the time series of annual
    CO<sub>2</sub> emissions by each country, with data reaching back to the
    mid-1700s for some countries.
</p>

<p>
    The emissions data is just as easy to import:
</p>

```python
df_emissions = pd.read_csv('./data/owid-co2-data.csv')
```

<h3 id="Combining the data sets">Combining the data sets</h3>

<p>
    It's typically beneficial to combine data from multiple sources into a new
    set of data. The relevant columns from the dataframes to extract are the
    year, CO<sub>2</sub> emissions and country centroid. To match the data
    across dataframes there needs to be a common unique identifier for each
    country.
</p>

<p>
    The <code>df_emissions</code> dataframe has an <code>iso_code</code> column
    that contains
    <a href="https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes">ISO
    3166-1 Alpha-3</a> country codes, while the <code>df_coords</code> dataframe
    has the slightly erroneously named <code>ISO3136</code> column containing
    the ISO 3166-1 Alpha-2 country codes. As is often the case, there already
    exists a Python library for performing an oddly specific task; in this case,
    mapping between Alpha-3 and Alpha-2 country codes can be performed with
    <a href="https://pypi.org/project/pycountry-convert/">pycountry-convert</a>.
    The main logic for creating the new dataframe is as follows:
</p>

```python
import pycountry_convert as pcc

data = []
for i, row in df_emissions.iterrows():
    store_data = False

    # Extract the ISO 3166-1 Alpha-3 country code
    alpha3 = row['iso_code']

    # Get latitude-longitude coordinates
    if str(alpha3) not in ['OWID_KOS', 'OWID_WRL', 'nan']:
        alpha2 = pcc.country_alpha3_to_country_alpha2(alpha3)
        try:
            lat = df_coords[df_coords['ISO3136']==alpha2]['LAT'].values[0]
            long = df_coords[df_coords['ISO3136']==alpha2]['LONG'].values[0]
            store_data = True
        except IndexError:
            store_data = False
    # Custom rule for Kosovo
    elif str(alpha3) == 'OWID_KOS':
        alpha2 = 'KOS'
        lat = 42.583333
        long = 21.0
        store_data = True
    
    year = row['year']
    CO2 = row['co2']

    if store_data:
        data.append([alpha2, lat, long, year, CO2])

df_data = pd.DataFrame(
    data,
    columns=['ISO', 'LAT', 'LONG', 'YEAR', 'CO2']
)
```

<h2 id="Calculating the emissions centre">Calculating the emissions centre</h2>

<h3 id="Converting between coordinate frames">Converting between coordinate frames</h3>

<p>
    To calculate the geospatial centre of emissions, it helps to be in the
    Cartesian coordinate frame. The conversion from spherical coordinates (e.g.
    latitude-longitude) to Cartesian coordinates is:
</p>

<p>
    $$\begin{aligned}
    x &= \cos(\text{lat}) \cos(\text{long}) \\
    y &= \cos(\text{lat}) \sin(\text{long}) \\ z &= \sin(\text{lat}).
    \end{aligned}$$
</p>

<p>
    To convert back to spherical coordinates:
</p>

<p>
    $$\begin{aligned}
    \text{lat} &= \arcsin(x) \\
    \text{long} &= {\arctan}{2}(y, x).
    \end{aligned}$$
</p>

<p>
    I've normalised both conversions to the radius of the earth.
</p>

<h3 id="Calculating the weighted mean">Calculating the weighted mean</h3>

<p>
    Calculating the weighted mean in Cartesian geometry is quite simple with the
    NumPy <a
    href="https://numpy.org/doc/stable/reference/generated/numpy.average.html"><code>average</code></a>
    function, which takes a <code>weights</code> argument. Here, we can
    independently take the average of a set of coordinates in a given direction,
    weighted by the CO<sub>2</sub> emissions.
</p>

<p>
    Creating the time series of weighted geospatial coordinate average
    CO<sub>2</sub> emissions is now just a case of looping through the years in
    the data set, converting each country's latitude-longitude coordinate to
    Cartesian, calculating the weighted mean in each Coordinate, then converting
    back to spherical coordinates.
</p>

```python
# Time series data
year_ts = sorted(df_data['YEAR'].unique())
lat_ts, long_ts = [], []

for i, year in enumerate(year_ts):
    # Get Cartesian coordinates and CO2 emissions of each country with data for
    # the current year
    df_year = df_data[data_data['YEAR']==year]
    x_vals, y_vals, z_vals, CO2_vals = [], [], [], []
    for _, row in df_year[['LAT', 'LONG', 'CO2', 'YEAR']].iterrows():
        x, y, z = convert_lat_long_to_cartesian(row['LAT'], row['LONG'])
        x_vals.append(x)
        y_vals.append(y)
        z_vals.append(z)
        CO2_vals.append(row['CO2'])
    
    avg_x, avg_y, avg_z = weighted_average_coordinates(
        x_vals, y_vals, z_vals, CO2_vals
    )
    
    avg_lat, avg_long = convert_cartesian_to_lat_long(avg_x, avg_y, avg_z)
    lat_ts.append(avg_lat)
    long_ts.append(avg_long)
```

<h2 id="The result">The result</h2>

<p>
    The time series can be visualised with <a
    href="https://matplotlib.org/">Matplotlib</a>'s <a
    href="https://matplotlib.org/stable/api/_as_gen/matplotlib.animation.FuncAnimation.html">FuncAnimation</a>
    class. After specifying a custom Matplotlib style and interpolating the data
    to make it less jittery (see the <a
    href="https://github.com/georgeholt1/geospatial_CO2">GitHub repo</a> for
    details) the following animation is generated (I recommend watching in full
    screen at high definition):
</p>

<iframe width="560" height="315" src="https://www.youtube.com/embed/TgX6_HdYA8s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>