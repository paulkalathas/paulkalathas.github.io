---
layout: post
title: Analyzing Active Covid Cases with Python
image: "/posts/coronavirus-2.jpg"
tags: [Python, Primes]
---

First let's start by importing all of the required modules needed for this exercise.  All of these might not be known at the start of your planning and mapping out you data analysis but the list can be added to later.  Let's get started

```ruby
#Installs

#Imported Modules
import datetime as dt
import urllib.request
import datetime
import calendar
import matplotlib.pyplot as plt
import seaborn as sns

# essential libraries
import json
import random
from urllib.request import urlopen

# storing and anaysis
import numpy as np
import pandas as pd

# visualization
import plotly.express as px
import plotly.graph_objs as go
import plotly.figure_factory as ff

# color pallette
cnf = '#393e46' # confirmed - grey
dth = '#ff2e63' # death - red
rec = '#21bf73' # recovered - cyan
act = '#fe9801' # active case - yellow

# converter
from pandas.plotting import register_matplotlib_converters
register_matplotlib_converters()   

# hide warnings
import warnings
warnings.filterwarnings('ignore')
```



