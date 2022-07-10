# The DC Recommender

## Intro

DataCamp is a great website for learning skills for data science. They have a lot of courses, mostly in R and Python, around all kinds of DS techniques, everything from manipulating time series data, to data viz courses, to different kinds of machine learning etc.

The courses are laid out in 'tracks', which group courses together under a more general topic. These can be 'Skill Tracks' or 'Career Tracks', and you can select any one track on the DC website to guide your learning. A side effect of this grouping is that some courses appear in multiple tracks. On completion of a track, you get a certificate, which can not only be shared on LinkedIn, but also makes you feel warm and fuzzy inside...

Ahem.. 

The issue I have identified is exemplified by DataCamp's 'Career Tracks', which are longer than the 'Skill Tracks' and cover a broader range of topics. By following a longer Career Track, due to the overlap with the shorter Skill Tracks, en route you may inadvertently complete the requisite material for multiple Skill Tracks (or be very close!). However, by not being enrolled on those Skill Tracks, you have no idea this is the case, and so you don't get those flashy DC certificate along the way despite you having done the required work - outrageous!

The goal is clear: work out a way of recommending courses based on how close they get you to finishing a track, so that you can rack up those treasured Track certificates as you learn.

The recommender should include some key ideas, such as:
- Course length (some courses are longer than others)
- How many tracks a course appears in (if it appears in more tracks, it brings you closer to multiple certificates)
- Some measure of how close a course gets you to completing a track (the closer, the better!)

Whilst DataCamp doesn't provide the above information directly, it is possible to infer by going through the various course pages on the website. It is also important to say that some tracks are organised into a logical progession of knowledge, which is particularly important with the more fundamental courses - any recommendation of a particular course may simply direct you to the appropriate track to follow in the original order.

### Approach


The DataCamp website contains over 50 tracks, which each contain multiple courses. It would take a long time to manually collect the data for this, so this should be scraped from the website where possible. All processing of the data should be as automated as possible. It would be great to get to a point where I could simply run a script and get up-to-date recommendations.

Whilst the traffic provided by scraping the website course pages is negligible for a company like DC, this should be kept to a minimum whilst testing and coding out of principle. Minimising repeated scraping of the same data will minimise traffic and save me time, so data should be saved / pickled along the way.

An additional thing to note: I keep all my DC course certificates in a Github repo. This was initially set up as a DC profile duplicate due to some public access issue on the DC website. That issue has since been resolved, so now I just use that repo as certificate storage, and is something that can be scraped to track my DC progress.

Right, enough talking, time to code.

## Imports


```python
import pickle
import pandas as pd
import undetected_chromedriver as uc
from bs4 import BeautifulSoup
from selenium.webdriver.chrome.options import Options
from typing import Tuple, List

# import requests
# from selenium import webdriver
# from selenium.webdriver.chrome.service import Service
# from webdriver_manager.chrome import ChromeDriverManager
```

## Scraping a track

DataCamp is already proving slightly troublesome. I could not just access the website's pages as HTML using `requests` as they use JavaScript rendering in the browser to return the required content.

Annoying, but not uncommon - Selenium is a fab package that renders a webpage for you, so problem solved? Alas, the default driver `webdriver` from Selenium doesn't work to access the page either, as the page request is redirected through Cloudflare, which seems to block bot access presumably for DDOS protection (I'm not a evil bot, honest!).

The solution? The `undetected_chromedriver` patch reconfigures a few settings and so does not trigger CloudFlare's bot-vision, and works a treat!


```python
# Set webpage to open in headless browser
options = Options()
options.headless = True
options.add_argument("--window-size=1920,1200")
# driver = webdriver.Chrome(options=options, service=Service(ChromeDriverManager().install()))
# driver = webdriver.Chrome(options=options)
driver = uc.Chrome(options=options)

# Set a test webpage to open
URL = "https://www.datacamp.com/tracks/time-series-with-python"

# Render page
driver.get(URL)
# Store the markup
page_source = driver.page_source
driver.quit()
# Parse with Beautiful Soup
soup = BeautifulSoup(page_source, "html.parser")
```

BeautifulSoup is a well-known library for parsing html code. Exploring the rendered html, I hit Ctrl+F to find the data I want and identify the tags/class names used for the titles, descriptions, and times for each course. Lets see what Beautiful Soup can do...


```python
course_name = soup.find_all('strong', class_ = 'css-1dbp6pz-TrackContentCard')
course_desc = soup.find_all('p', class_ = 'css-r9ojyg-TrackContentCard')
course_time = soup.find_all('p', class_ = 'css-1jr04uj-TrackContentCard')

course_name = [x.text for x in course_name]
course_desc = [x.text for x in course_desc]
course_time = [x.text for x in course_time]

course_name, course_desc, course_time
```




    (['Manipulating Time Series Data in Python',
      'Time Series Analysis in Python',
      'Visualizing Time Series Data in Python',
      'ARIMA Models in Python',
      'Machine Learning for Time Series Data in Python'],
     ["In this course you'll learn the basics of working with time series data.",
      "In this course you'll learn the basics of analyzing time series data.",
      'Visualize seasonality, trends and other patterns in your time series data. ',
      'Learn about ARIMA models in Python and become an expert in time series analysis.',
      'This course focuses on feature engineering and machine learning for time series data. '],
     ['4 hours', '4 hours', '4 hours', '4 hours', '4 hours'])



Beautiful! This lets me pull just the info I need in a neat way. Let's get this into a function so I can just plug in a URL and get what I want. It makes sense to split this into 2 functions so I can also just retrieve the HTML of other pages if required.


```python
def get_html(dc_url) -> BeautifulSoup:
    """
    Get the HTML of a page

    Parameters
    ----------
    dc_url : str
        URL of a page

    Returns
    -------
    BeautifulSoup
        BeautifulSoup object of the page
    """
    # Set webpage to open in headless browser
    options = Options()
    options.headless = True
    options.add_argument("--window-size=1920,1200")
    # New browser instance started each time function is called to avoid errors from server limits - don't abuse this!
    driver = uc.Chrome(options=options)
    
    # Render page and save page source
    driver.get(dc_url)
    page_source = driver.page_source
    driver.quit()
    
    # Parse with Beautiful Soup
    soup = BeautifulSoup(page_source, "html.parser")
    return soup


def get_courses(dc_url) -> list:
    """
    Get list of courses from tracks on datacamp.com

    Parameters
    ----------
    dc_url : str
        URL of datacamp.com track page

    Returns
    -------
    list
        List of courses
    """
    soup = get_html(dc_url)
    # Get the courses info
    course_names = soup.find_all('strong', class_ = 'css-1dbp6pz-TrackContentCard')
    course_names = [x.text for x in course_names]
    course_descs = soup.find_all('p', class_ = 'css-r9ojyg-TrackContentCard')
    course_descs = [x.text.strip() for x in course_descs]   # Strip whitespace from description
    course_times = soup.find_all('p', class_ = 'css-1jr04uj-TrackContentCard')
    course_times = [x.text for x in course_times]
    
    # Combine into a list of tuples
    courses = list(zip(course_names, course_descs, course_times))
    return courses
```

Let's give it a spin on a random track page


```python
test_track = get_courses("https://www.datacamp.com/tracks/deep-learning-in-python")
test_track
```




    [('Introduction to Deep Learning in Python',
      'Learn the fundamentals of neural networks and how to build deep learning models using Keras 2.0.',
      '4 hours'),
     ('Introduction to TensorFlow in Python',
      'Learn the fundamentals of neural networks and how to build deep learning models using TensorFlow.',
      '4 hours'),
     ('Introduction to Deep Learning with PyTorch',
      'Learn to create deep learning models with the PyTorch library.',
      '4 hours'),
     ('Introduction to Deep Learning with Keras',
      'Learn to start developing deep learning models with Keras.',
      '4 hours'),
     ('Advanced Deep Learning with Keras',
      'Build multiple-input and multiple-output deep learning models using Keras.',
      '4 hours')]



Great, the function works fine.

Next up, I need to get a list of the tracks and their web addresses. I could compile this manually as there are far fewer tracks than courses, but lets see if this can be scraped also.

## Scraping the tracks lists


```python
# Get the html of the skill tracks page
st_url = "https://www.datacamp.com/tracks/skill"
st_soup = get_html(st_url)

# Get the names of the skill tracks
# track_names = st_soup.find_all('h3', class_ = 'css-so6h6r-TrackCard')     # The class is not consistent between reloads, use heading 3 only instead
track_names = st_soup.find_all('h3')[:53]
track_names = [x.text.strip() for x in track_names]

# Get the technologies used in the skill tracks
track_techs = st_soup.find_all('title')[1:54]
track_techs = [x.text.strip() for x in track_techs]

# To get the web addresses, using the class tags doesn't work, as some of the weblinks have different html classes
# Instead, first find all hrefs from the page where the reference begins with '/tracks/'
track_urls = st_soup.find_all('a', href = lambda x: x and x.startswith('/tracks/'))
track_urls = [x['href'] for x in track_urls]
# The urls are relative, so need to add the base url
track_urls = ["https://www.datacamp.com" + x for x in track_urls]
# Remove links not pointing to a skill track
track_urls = track_urls[8:-2]
track_urls

# Finally, combine the names, technologies, and urls into a list of tuples
skill_tracks = list(zip(track_names, track_techs, track_urls))
skill_tracks
```




    [('R Programming', 'R', 'https://www.datacamp.com/tracks/r-programming'),
     ('Importing & Cleaning Data',
      'R',
      'https://www.datacamp.com/tracks/importing-cleaning-data-with-r'),
     ('Data Visualization',
      'R',
      'https://www.datacamp.com/tracks/data-visualization-with-r'),
     ('Data Manipulation',
      'R',
      'https://www.datacamp.com/tracks/data-manipulation-with-r'),
     ('Statistics Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/learn-statistics-with-r'),
     ('Python Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/python-fundamentals'),
     ('Importing & Cleaning Data',
      'Python',
      'https://www.datacamp.com/tracks/importing-cleaning-data-with-python'),
     ('Data Manipulation',
      'Python',
      'https://www.datacamp.com/tracks/data-manipulation-with-python'),
     ('Time Series', 'R', 'https://www.datacamp.com/tracks/time-series-with-r'),
     ('Applied Finance',
      'R',
      'https://www.datacamp.com/tracks/applied-finance-in-r'),
     ('Finance Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-r'),
     ('Machine Learning Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/machine-learning-fundamentals'),
     ('Machine Learning Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/machine-learning-fundamentals-with-python'),
     ('Text Mining', 'R', 'https://www.datacamp.com/tracks/text-mining-with-r'),
     ('Spatial Data', 'R', 'https://www.datacamp.com/tracks/spatial-data-with-r'),
     ('Shiny Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/shiny-fundamentals-with-r'),
     ('Time Series',
      'Python',
      'https://www.datacamp.com/tracks/time-series-with-python'),
     ('Big Data', 'R', 'https://www.datacamp.com/tracks/big-data-with-r'),
     ('Tidyverse Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/tidyverse-fundamentals'),
     ('Network Analysis',
      'R',
      'https://www.datacamp.com/tracks/analyzing-networks-with-r'),
     ('Supervised Machine Learning',
      'R',
      'https://www.datacamp.com/tracks/supervised-machine-learning-in-r'),
     ('Interactive Data Visualization',
      'R',
      'https://www.datacamp.com/tracks/interactive-data-visualization-in-r'),
     ('SQL Fundamentals',
      'SQL',
      'https://www.datacamp.com/tracks/sql-fundamentals'),
     ('Statistics Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/statistics-fundamentals-with-python'),
     ('Unsupervised Machine Learning',
      'R',
      'https://www.datacamp.com/tracks/unsupervised-machine-learning-with-r'),
     ('Marketing Analytics',
      'R',
      'https://www.datacamp.com/tracks/marketing-analytics-with-r'),
     ('Statistical Inference',
      'R',
      'https://www.datacamp.com/tracks/statistical-inference-with-r'),
     ('Data Visualization',
      'Python',
      'https://www.datacamp.com/tracks/data-visualization-with-python'),
     ('Intermediate Tidyverse Toolbox',
      'R',
      'https://www.datacamp.com/tracks/intermediate-tidyverse-toolbox'),
     ('SQL Server Fundamentals',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-fundamentals'),
     ('Image Processing',
      'Python',
      'https://www.datacamp.com/tracks/image-processing'),
     ('Spreadsheet Fundamentals',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/spreadsheet-fundamentals'),
     ('Python Toolbox',
      'Python',
      'https://www.datacamp.com/tracks/python-toolbox'),
     ('Analyzing Genomic Data',
      'R',
      'https://www.datacamp.com/tracks/analyzing-genomic-data-in-r'),
     ('Big Data with PySpark',
      'Python',
      'https://www.datacamp.com/tracks/big-data-with-pyspark'),
     ('Python Programming',
      'Python',
      'https://www.datacamp.com/tracks/python-programming'),
     ('Data Skills for Business',
      'Theory',
      'https://www.datacamp.com/tracks/foundational-data-skills-for-business-leaders'),
     ('Marketing Analytics',
      'Python',
      'https://www.datacamp.com/tracks/marketing-analytics-with-python'),
     ('SQL for Business Analysts',
      'SQL',
      'https://www.datacamp.com/tracks/sql-for-business-analysts'),
     ('SQL Server for Database Administrators',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-for-database-administrators'),
     ('SQL for Database Administrators',
      'SQL',
      'https://www.datacamp.com/tracks/sql-for-database-administrators'),
     ('Intermediate Spreadsheets',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/intermediate-spreadsheets'),
     ('Deep Learning',
      'Python',
      'https://www.datacamp.com/tracks/deep-learning-in-python'),
     ('Data Literacy Fundamentals',
      'Theory',
      'https://www.datacamp.com/tracks/data-literacy-fundamentals'),
     ('Natural Language Processing',
      'Python',
      'https://www.datacamp.com/tracks/natural-language-processing-in-python'),
     ('Deep Learning for NLP',
      'Python',
      'https://www.datacamp.com/tracks/deep-learning-for-nlp-in-python'),
     ('Finance Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-python'),
     ('Applied Finance',
      'Python',
      'https://www.datacamp.com/tracks/applied-finance-in-python'),
     ('Finance Fundamentals',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-spreadsheets'),
     ('Tableau Fundamentals',
      'Tableau',
      'https://www.datacamp.com/tracks/tableau-fundamentals'),
     ('Power BI Fundamentals',
      'Power BI',
      'https://www.datacamp.com/tracks/power-bi-fundamentals'),
     ('Data Skills for Business',
      'Theory',
      'https://www.datacamp.com/tracks/foundational-data-skills-for-business-leaders'),
     ('Data Literacy Fundamentals',
      'Theory',
      'https://www.datacamp.com/tracks/data-literacy-fundamentals')]



This worked, but required some literal indexing to remove irrelevant data. Not the greatest solution, but functional for now. If in future more tracks get added to DataCamp, this indexing may need re-examining, particularly if the layout of the page changes such that the number of tags to ignore from the beginning and end changes.
Let's see if we can get the career tracks as well.


```python
# Get the html of the career tracks page
ct_url = "https://www.datacamp.com/tracks/career"
ct_soup = get_html(ct_url)

# Get the names of the career tracks
track_names = ct_soup.find_all('h3', class_ = 'css-so6h6r-TrackCard')
track_names = [x.text.strip() for x in track_names]

# Get the technologies of the career tracks
track_techs = ct_soup.find_all('title')[1:15]
track_techs = [x.text.strip() for x in track_techs]

# To get the web addresses, using the class tags doesn't work, as some of the weblinks have different html classes
# Instead, first find all hrefs from the page where the reference begins with '/tracks/'
track_urls = ct_soup.find_all('a', href = lambda x: x and x.startswith('/tracks/'))
track_urls = [x['href'] for x in track_urls]
# The urls are relative, so need to add the base url
track_urls = ["https://www.datacamp.com" + x for x in track_urls]
# Remove links not pointing to a career track
track_urls = track_urls[8:-2]
track_urls

# Finally, combine the names, technologies and urls into a list of tuples
career_tracks = list(zip(track_names, track_techs, track_urls))

career_tracks
```




    [('Data Analyst', 'R', 'https://www.datacamp.com/tracks/data-analyst-with-r'),
     ('Data Scientist',
      'R',
      'https://www.datacamp.com/tracks/data-scientist-with-r'),
     ('Data Analyst',
      'Python',
      'https://www.datacamp.com/tracks/data-analyst-with-python'),
     ('Data Scientist',
      'Python',
      'https://www.datacamp.com/tracks/data-scientist-with-python'),
     ('Python Programmer',
      'Python',
      'https://www.datacamp.com/tracks/python-programmer'),
     ('R Programmer', 'R', 'https://www.datacamp.com/tracks/r-programmer'),
     ('Quantitative Analyst',
      'R',
      'https://www.datacamp.com/tracks/quantitative-analyst-with-r'),
     ('Statistician', 'R', 'https://www.datacamp.com/tracks/statistician-with-r'),
     ('Machine Learning Scientist',
      'R',
      'https://www.datacamp.com/tracks/machine-learning-scientist-with-r'),
     ('Machine Learning Scientist',
      'Python',
      'https://www.datacamp.com/tracks/machine-learning-scientist-with-python'),
     ('Data Engineer',
      'Python',
      'https://www.datacamp.com/tracks/data-engineer-with-python'),
     ('SQL Server Developer',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-developer'),
     ('Data Analyst',
      'Power BI',
      'https://www.datacamp.com/tracks/data-analyst-in-power-bi'),
     ('Data Analyst',
      'SQL',
      'https://www.datacamp.com/tracks/data-analyst-in-sql')]




```python
# Combine the skill and career tracks into one list
all_tracks = skill_tracks + career_tracks
```

Once scraped, pickle to avoid re-running code. As I come back to this, I can then just read in the pickle and avoid scraping again.


```python
# Save to a text file
with open('all_tracks.txt', 'w') as f:
    for track in all_tracks:
        # Seperate each element in the tuple with ', '
        f.write(', '.join(track) + '\n')
```


```python
# Read the text file back in
with open('all_tracks.txt', 'r') as f:
    all_tracks = f.readlines()
    all_tracks = [x.split(', ') for x in all_tracks]
all_tracks
```




    [['R Programming', 'R', 'https://www.datacamp.com/tracks/r-programming\n'],
     ['Importing & Cleaning Data',
      'R',
      'https://www.datacamp.com/tracks/importing-cleaning-data-with-r\n'],
     ['Data Visualization',
      'R',
      'https://www.datacamp.com/tracks/data-visualization-with-r\n'],
     ['Data Manipulation',
      'R',
      'https://www.datacamp.com/tracks/data-manipulation-with-r\n'],
     ['Statistics Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/learn-statistics-with-r\n'],
     ['Python Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/python-fundamentals\n'],
     ['Importing & Cleaning Data',
      'Python',
      'https://www.datacamp.com/tracks/importing-cleaning-data-with-python\n'],
     ['Data Manipulation',
      'Python',
      'https://www.datacamp.com/tracks/data-manipulation-with-python\n'],
     ['Time Series', 'R', 'https://www.datacamp.com/tracks/time-series-with-r\n'],
     ['Applied Finance',
      'R',
      'https://www.datacamp.com/tracks/applied-finance-in-r\n'],
     ['Finance Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-r\n'],
     ['Machine Learning Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/machine-learning-fundamentals\n'],
     ['Machine Learning Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/machine-learning-fundamentals-with-python\n'],
     ['Text Mining', 'R', 'https://www.datacamp.com/tracks/text-mining-with-r\n'],
     ['Spatial Data',
      'R',
      'https://www.datacamp.com/tracks/spatial-data-with-r\n'],
     ['Shiny Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/shiny-fundamentals-with-r\n'],
     ['Time Series',
      'Python',
      'https://www.datacamp.com/tracks/time-series-with-python\n'],
     ['Big Data', 'R', 'https://www.datacamp.com/tracks/big-data-with-r\n'],
     ['Tidyverse Fundamentals',
      'R',
      'https://www.datacamp.com/tracks/tidyverse-fundamentals\n'],
     ['Network Analysis',
      'R',
      'https://www.datacamp.com/tracks/analyzing-networks-with-r\n'],
     ['Supervised Machine Learning',
      'R',
      'https://www.datacamp.com/tracks/supervised-machine-learning-in-r\n'],
     ['Interactive Data Visualization',
      'R',
      'https://www.datacamp.com/tracks/interactive-data-visualization-in-r\n'],
     ['SQL Fundamentals',
      'SQL',
      'https://www.datacamp.com/tracks/sql-fundamentals\n'],
     ['Statistics Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/statistics-fundamentals-with-python\n'],
     ['Unsupervised Machine Learning',
      'R',
      'https://www.datacamp.com/tracks/unsupervised-machine-learning-with-r\n'],
     ['Marketing Analytics',
      'R',
      'https://www.datacamp.com/tracks/marketing-analytics-with-r\n'],
     ['Statistical Inference',
      'R',
      'https://www.datacamp.com/tracks/statistical-inference-with-r\n'],
     ['Data Visualization',
      'Python',
      'https://www.datacamp.com/tracks/data-visualization-with-python\n'],
     ['Intermediate Tidyverse Toolbox',
      'R',
      'https://www.datacamp.com/tracks/intermediate-tidyverse-toolbox\n'],
     ['SQL Server Fundamentals',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-fundamentals\n'],
     ['Image Processing',
      'Python',
      'https://www.datacamp.com/tracks/image-processing\n'],
     ['Spreadsheet Fundamentals',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/spreadsheet-fundamentals\n'],
     ['Python Toolbox',
      'Python',
      'https://www.datacamp.com/tracks/python-toolbox\n'],
     ['Analyzing Genomic Data',
      'R',
      'https://www.datacamp.com/tracks/analyzing-genomic-data-in-r\n'],
     ['Big Data with PySpark',
      'Python',
      'https://www.datacamp.com/tracks/big-data-with-pyspark\n'],
     ['Python Programming',
      'Python',
      'https://www.datacamp.com/tracks/python-programming\n'],
     ['Data Skills for Business',
      'Theory',
      'https://www.datacamp.com/tracks/foundational-data-skills-for-business-leaders\n'],
     ['Marketing Analytics',
      'Python',
      'https://www.datacamp.com/tracks/marketing-analytics-with-python\n'],
     ['SQL for Business Analysts',
      'SQL',
      'https://www.datacamp.com/tracks/sql-for-business-analysts\n'],
     ['SQL Server for Database Administrators',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-for-database-administrators\n'],
     ['SQL for Database Administrators',
      'SQL',
      'https://www.datacamp.com/tracks/sql-for-database-administrators\n'],
     ['Intermediate Spreadsheets',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/intermediate-spreadsheets\n'],
     ['Deep Learning',
      'Python',
      'https://www.datacamp.com/tracks/deep-learning-in-python\n'],
     ['Data Literacy Fundamentals',
      'Theory',
      'https://www.datacamp.com/tracks/data-literacy-fundamentals\n'],
     ['Natural Language Processing',
      'Python',
      'https://www.datacamp.com/tracks/natural-language-processing-in-python\n'],
     ['Deep Learning for NLP',
      'Python',
      'https://www.datacamp.com/tracks/deep-learning-for-nlp-in-python\n'],
     ['Finance Fundamentals',
      'Python',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-python\n'],
     ['Applied Finance',
      'Python',
      'https://www.datacamp.com/tracks/applied-finance-in-python\n'],
     ['Finance Fundamentals',
      'Spreadsheet',
      'https://www.datacamp.com/tracks/finance-fundamentals-in-spreadsheets\n'],
     ['Tableau Fundamentals',
      'Tableau',
      'https://www.datacamp.com/tracks/tableau-fundamentals\n'],
     ['Power BI Fundamentals',
      'Power BI',
      'https://www.datacamp.com/tracks/power-bi-fundamentals\n'],
     ['Data Skills for Business',
      'Theory',
      'https://www.datacamp.com/tracks/foundational-data-skills-for-business-leaders\n'],
     ['Data Literacy Fundamentals',
      'Theory',
      'https://www.datacamp.com/tracks/data-literacy-fundamentals\n'],
     ['Data Analyst',
      'R',
      'https://www.datacamp.com/tracks/data-analyst-with-r\n'],
     ['Data Scientist',
      'R',
      'https://www.datacamp.com/tracks/data-scientist-with-r\n'],
     ['Data Analyst',
      'Python',
      'https://www.datacamp.com/tracks/data-analyst-with-python\n'],
     ['Data Scientist',
      'Python',
      'https://www.datacamp.com/tracks/data-scientist-with-python\n'],
     ['Python Programmer',
      'Python',
      'https://www.datacamp.com/tracks/python-programmer\n'],
     ['R Programmer', 'R', 'https://www.datacamp.com/tracks/r-programmer\n'],
     ['Quantitative Analyst',
      'R',
      'https://www.datacamp.com/tracks/quantitative-analyst-with-r\n'],
     ['Statistician',
      'R',
      'https://www.datacamp.com/tracks/statistician-with-r\n'],
     ['Machine Learning Scientist',
      'R',
      'https://www.datacamp.com/tracks/machine-learning-scientist-with-r\n'],
     ['Machine Learning Scientist',
      'Python',
      'https://www.datacamp.com/tracks/machine-learning-scientist-with-python\n'],
     ['Data Engineer',
      'Python',
      'https://www.datacamp.com/tracks/data-engineer-with-python\n'],
     ['SQL Server Developer',
      'SQL',
      'https://www.datacamp.com/tracks/sql-server-developer\n'],
     ['Data Analyst',
      'Power BI',
      'https://www.datacamp.com/tracks/data-analyst-in-power-bi\n'],
     ['Data Analyst',
      'SQL',
      'https://www.datacamp.com/tracks/data-analyst-in-sql\n']]



## Scrape all courses for all tracks

Right, with a list of tracks, techs, and their web addresses, plus a method to scrape the course info from a track webpage, we should be able to get the course list for every track.


```python
def read_track_generator(file_object):
    """
    Generator function to read a text file line by line
    
        Parameters
        ----------
        file_object : file
            File object to read
    """
    
    # Loop indefinitely until end of the file
    while True:
        # Read a track name and web address from the file
        data = file_object.readline()
        # Break if this is the end of the file
        if not data:
            break

        # Change data object to list
        data = data.split(",")
        # Remove newline character and spaces
        data = [x.strip() for x in data]

        # Yield the line of data
        yield data
```

Let the scraping commence!

As a side note: had to do a little bit of QA, just to make sure the courses read in correctly. If any tracks are empty, then that dictionary entry is deleted and we re-scrape it. In practice, this happened seemingly at random to just a couple of courses, and may be from datacamp sometimes changing the class names of objects when generating the page. This is slightly strange behaviour, and indicates that I shouldn't rely on HTML class names in the future, but for now this is a minor issue and easily managed.


```python
complete_tracks = dict()
# Loop with check variable of `read_tracks` to check if any tracks haven't read correctly
read_tracks = 1
while read_tracks > 0:

    # Open a connection to the file
    with open('all_tracks.txt') as f:

        # Create a generator object for the file
        gen_file = read_track_generator(f)

        # Loop through the generator object
        while True:
            try:
                # Read the next track's details
                this_track = next(gen_file)
                # Break if this is the end of the file
                if not this_track:
                    break
            except:
                break

            # Add the technology to the key name
            this_track[0] = this_track[0] + ", " + this_track[1]
            
            # Check if track key already exists in the dictionary
            if this_track[0] in complete_tracks:
                continue        # Skip iteration if key already exists, allows for resuming a broken loop without re-executing on beginning of file
                # # If it already exists, create a new key by adding the last 3 words of the url to the key
                # this_track[0] = this_track[0] + " - " + " ".join(this_track[2].split("-")[-2:])

            # Get the courses for this track
            this_courses = get_courses(this_track[2])
            # Convert this_courses to a dictionary with the first element as the key and the second and third elements as the value
            this_courses = {x[0]: (x[1], x[2]) for x in this_courses}
            
            # Add the track to the dictionary
            complete_tracks[this_track[0]] = this_courses
            # Print progress
            print(this_track[0])
    read_tracks = 0

    # Delete tracks with no courses read
    for track in complete_tracks.copy():
        if len(complete_tracks[track]) == 0:
            print('>>', track)
            del complete_tracks[track]
            read_tracks += 1
```


```python
# Save to a pickle file
with open('complete_tracks.pickle', 'wb') as f:
    pickle.dump(complete_tracks, f)
```

Scraping success! On to processing the data!

## Processing


```python
# Import complete_tracks from the pickle file
with open('complete_tracks.pickle', 'rb') as f:
    complete_tracks = pickle.load(f)
```

I'd like to have some strategy for completing these courses. I want to get through as many tracks as I can and as quickly as possible. My focus is currently on Python, so I want to get those done first, as well as some Power BI courses. However, it is good to know all of my options, so I shouldn't exclude the other technologies. It may intially be useful to just get an idea of how the courses are split between the technologies.


```python
# Get a list of technologies from the dictionary, where the term after the comma is the technology
technologies = {x.split(", ")[1] for x in complete_tracks.keys()}
technologies

# Show all courses for a given technology
def show_courses(tech):
    """
    Prints all courses names for a given technology
    
        Parameters
        ----------
        tech : str
            Technology to search for
    """
    print("\nCourses for " + tech + ":")
    # Loop through the dictionary
    for key in complete_tracks:
        # Check if the technology is in the key
        if tech in key:
            # Print the key and the courses
            print('\t', key)
            for course in complete_tracks[key]:
                print('\t'*2, course)
            print()

for tech in technologies:
    show_courses(tech)
```

    
    Courses for Theory:
    	 Data Skills for Business, Theory
    		 Data Science for Business
    		 Introduction to Statistics in Spreadsheets
    		 Machine Learning for Business
    		 Data-Driven Decision Making for Business
    		 Marketing Analytics for Business
    		 AI Fundamentals
    		 Introduction to Data Science in Python
    
    	 Data Literacy Fundamentals, Theory
    		 Data Science for Everyone
    		 Machine Learning for Everyone
    		 Data Visualization for Everyone
    		 Data Engineering for Everyone
    		 Cloud Computing for Everyone
    
    
    Courses for Python:
    	 Python Fundamentals, Python
    		 Introduction to Python
    		 Intermediate Python
    		 Python Data Science Toolbox (Part 1)
    		 Python Data Science Toolbox (Part 2)
    
    	 Importing & Cleaning Data, Python
    		 Introduction to Importing Data in Python
    		 Intermediate Importing Data in Python
    		 Cleaning Data in Python
    		 Reshaping Data with pandas
    
    	 Data Manipulation, Python
    		 Data Manipulation with pandas
    		 Joining Data with pandas
    		 Analyzing Police Activity with pandas
    		 Introduction to Databases in Python
    
    	 Machine Learning Fundamentals, Python
    		 Supervised Learning with scikit-learn
    		 Unsupervised Learning in Python
    		 Linear Classifiers in Python
    		 Introduction to Deep Learning in Python
    
    	 Time Series, Python
    		 Manipulating Time Series Data in Python
    		 Time Series Analysis in Python
    		 Visualizing Time Series Data in Python
    		 ARIMA Models in Python
    		 Machine Learning for Time Series Data in Python
    
    	 Statistics Fundamentals, Python
    		 Introduction to Statistics in Python
    		 Introduction to Regression with statsmodels in Python
    		 Intermediate Regression with statsmodels in Python
    		 Sampling in Python
    		 Hypothesis Testing in Python
    
    	 Data Visualization, Python
    		 Introduction to Data Visualization with Matplotlib
    		 Introduction to Data Visualization with Seaborn
    		 Improving Your Data Visualizations in Python
    		 Visualizing Geospatial Data in Python
    
    	 Image Processing, Python
    		 Image Processing in Python
    		 Biomedical Image Analysis in Python
    		 Image Processing with Keras in Python
    
    	 Python Toolbox, Python
    		 Dealing with Missing Data in Python
    		 Working with Dates and Times in Python
    		 Regular Expressions in Python
    		 Writing Efficient Python Code
    
    	 Big Data with PySpark, Python
    		 Introduction to PySpark
    		 Big Data Fundamentals with PySpark
    		 Cleaning Data with PySpark
    		 Feature Engineering with PySpark
    		 Machine Learning with PySpark
    		 Building Recommendation Engines with PySpark
    
    	 Python Programming, Python
    		 Writing Efficient Python Code
    		 Writing Efficient Code with pandas
    		 Writing Functions in Python
    		 Software Engineering for Data Scientists in Python
    		 Unit Testing for Data Science in Python
    		 Object-Oriented Programming in Python
    
    	 Marketing Analytics, Python
    		 Analyzing Marketing Campaigns with pandas
    		 Analyzing Social Media Data in Python
    		 Market Basket Analysis in Python
    		 Machine Learning for Marketing in Python
    		 Customer Segmentation in Python
    		 Marketing Analytics: Predicting Customer Churn in Python
    		 Customer Analytics and A/B Testing in Python
    
    	 Deep Learning, Python
    		 Introduction to Deep Learning in Python
    		 Introduction to TensorFlow in Python
    		 Introduction to Deep Learning with PyTorch
    		 Introduction to Deep Learning with Keras
    		 Advanced Deep Learning with Keras
    
    	 Natural Language Processing, Python
    		 Introduction to Natural Language Processing in Python
    		 Sentiment Analysis in Python
    		 Building Chatbots in Python
    		 Advanced NLP with spaCy
    		 Spoken Language Processing in Python
    		 Feature Engineering for NLP in Python
    
    	 Deep Learning for NLP, Python
    		 Recurrent Neural Networks for Language Modeling in Python
    		 Machine Translation in Python
    		 Natural Language Generation in Python
    
    	 Finance Fundamentals, Python
    		 Introduction to Python for Finance
    		 Intermediate Python for Finance
    		 Introduction to Financial Concepts in Python
    		 Manipulating Time Series Data in Python
    		 Importing and Managing Financial Data in Python
    		 Introduction to Portfolio Analysis in Python
    
    	 Applied Finance, Python
    		 Introduction to Portfolio Risk Management in Python
    		 Quantitative Risk Management in Python
    		 Credit Risk Modeling in Python
    		 GARCH Models in Python
    
    	 Data Analyst, Python
    		 Introduction to Data Science in Python
    		 Intermediate Python
    		 Data Manipulation with pandas
    		 Joining Data with pandas
    		 Introduction to Data Visualization with Seaborn
    		 Data Manipulation with Python
    		 Exploratory Data Analysis in Python
    		 Introduction to Statistics in Python
    
    	 Data Scientist, Python
    		 Introduction to Python
    		 Intermediate Python
    		 Investigating Netflix Movies and Guest Stars in The Office
    		 Data Manipulation with pandas
    		 Joining Data with pandas
    		 The GitHub History of the Scala Language
    		 Introduction to Data Visualization with Matplotlib
    		 Introduction to Data Visualization with Seaborn
    		 Introduction to NumPy
    		 Python Data Science Toolbox (Part 1)
    		 The Android App Market on Google Play
    		 Python Data Science Toolbox (Part 2)
    		 Intermediate Data Visualization with Seaborn
    		 A Visual History of Nobel Prize Winners
    		 Data Manipulation with Python
    		 Introduction to Importing Data in Python
    		 Intermediate Importing Data in Python
    		 Cleaning Data in Python
    		 Working with Dates and Times in Python
    		 Importing & Cleaning Data with Python
    		 Writing Functions in Python
    		 Python Programming
    		 Exploratory Data Analysis in Python
    		 Analyzing Police Activity with pandas
    		 Introduction to Statistics in Python
    		 Introduction to Regression with statsmodels in Python
    		 Sampling in Python
    		 Hypothesis Testing in Python
    		 Dr. Semmelweis and the Discovery of Handwashing
    		 Supervised Learning with scikit-learn
    		 Predicting Credit Card Approvals
    
    	 Python Programmer, Python
    		 Introduction to Data Science in Python
    		 Data Manipulation with pandas
    		 Python Data Science Toolbox (Part 1)
    		 Python Data Science Toolbox (Part 2)
    		 Data Types for Data Science in Python
    		 Writing Efficient Python Code
    		 Working with Dates and Times in Python
    		 Regular Expressions in Python
    		 Web Scraping in Python
    		 Writing Functions in Python
    		 Introduction to Shell
    		 Software Engineering for Data Scientists in Python
    		 Developing Python Packages
    		 Unit Testing for Data Science in Python
    		 Object-Oriented Programming in Python
    
    	 Machine Learning Scientist, Python
    		 Supervised Learning with scikit-learn
    		 Unsupervised Learning in Python
    		 Linear Classifiers in Python
    		 Machine Learning with Tree-Based Models in Python
    		 Extreme Gradient Boosting with XGBoost
    		 Cluster Analysis in Python
    		 Dimensionality Reduction in Python
    		 Preprocessing for Machine Learning in Python
    		 Machine Learning for Time Series Data in Python
    		 Feature Engineering for Machine Learning in Python
    		 Model Validation in Python
    		 Machine Learning Fundamentals in Python
    		 Introduction to Natural Language Processing in Python
    		 Feature Engineering for NLP in Python
    		 Introduction to TensorFlow in Python
    		 Introduction to Deep Learning in Python
    		 Introduction to Deep Learning with Keras
    		 Advanced Deep Learning with Keras
    		 Image Processing in Python
    		 Image Processing with Keras in Python
    		 Hyperparameter Tuning in Python
    		 Introduction to PySpark
    		 Machine Learning with PySpark
    
    	 Data Engineer, Python
    		 Data Engineering for Everyone
    		 Python Programming
    		 Introduction to Data Engineering
    		 Streamlined Data Ingestion with pandas
    		 Writing Efficient Python Code
    		 Writing Functions in Python
    		 Introduction to Shell
    		 Data Processing in Shell
    		 Introduction to Bash Scripting
    		 Unit Testing for Data Science in Python
    		 Object-Oriented Programming in Python
    		 Introduction to Airflow in Python
    		 Introduction to PySpark
    		 Introduction to AWS Boto in Python
    		 Data Analysis in SQL (PostgreSQL)
    		 Introduction to Relational Databases in SQL
    		 Database Design
    		 Introduction to Scala
    		 Big Data Fundamentals with PySpark
    
    
    Courses for Tableau:
    	 Tableau Fundamentals, Tableau
    		 Introduction to Tableau
    		 Analyzing Data in Tableau
    		 Creating Dashboards in Tableau
    		 Case Study: Analyzing Customer Churn in Tableau
    		 Connecting Data in Tableau
    
    
    Courses for Power BI:
    	 Power BI Fundamentals, Power BI
    		 Introduction to Power BI
    		 Introduction to DAX in Power BI
    		 Data Visualization in Power BI
    		 Case Study: Analyzing Job Market Data in Power BI
    		 Data Preparation in Power BI
    		 Data Modeling in Power BI
    
    	 Data Analyst, Power BI
    		 Introduction to Power BI
    		 Introduction to DAX in Power BI
    		 Data Visualization in Power BI
    		 Case Study: Analyzing Job Market Data in Power BI
    		 Case Study: Analyzing Customer Churn in Power BI
    		 Data Preparation in Power BI
    		 Data Transformation in Power BI
    		 Data Modeling in Power BI
    		 Intermediate Data Modeling in Power BI
    		 DAX Functions in Power BI
    		 Intermediate DAX in Power BI
    		 User-Oriented Design in Power BI
    		 Exploratory Data Analysis in Power BI
    		 Trend Analysis in Power BI
    		 Reports in Power BI
    		 Report Design in Power BI
    		 Data Connections in Power BI
    		 Deploying and Maintaining Assets in Power BI
    
    
    Courses for R:
    	 R Programming, R
    		 Introduction to R
    		 Intermediate R
    		 Writing Efficient R Code
    		 Introduction to Writing Functions in R
    		 Object-Oriented Programming with S3 and R6 in R
    
    	 Importing & Cleaning Data, R
    		 Introduction to Importing Data in R
    		 Intermediate Importing Data in R
    		 Cleaning Data in R
    		 Reshaping Data with tidyr
    		 Exploring the NYC Airbnb Market
    
    	 Data Visualization, R
    		 Introduction to Data Visualization with ggplot2
    		 Intermediate Data Visualization with ggplot2
    		 Visualization Best Practices in R
    
    	 Data Manipulation, R
    		 Data Manipulation with dplyr 
    		 Joining Data with dplyr
    		 Case Study: Exploratory Data Analysis in R
    		 Data Manipulation with data.table in R
    		 Joining Data with data.table in R
    
    	 Statistics Fundamentals, R
    		 Introduction to Statistics in R
    		 Introduction to Regression in R
    		 Intermediate Regression in R
    		 Sampling in R
    		 Hypothesis Testing in R
    
    	 Time Series, R
    		 Manipulating Time Series Data with xts and zoo in R
    		 Time Series Analysis in R
    		 ARIMA Models in R
    		 Forecasting in R
    		 Visualizing Time Series Data in R
    		 Case Studies: Manipulating Time Series Data in R
    
    	 Applied Finance, R
    		 Quantitative Risk Management in R
    		 Equity Valuation in R
    		 Life Insurance Products Valuation in R
    		 Bond Valuation and Analysis in R
    		 Financial Trading in R
    		 Credit Risk Modeling in R
    		 GARCH Models in R
    
    	 Finance Fundamentals, R
    		 Introduction to R for Finance
    		 Intermediate R for Finance
    		 Manipulating Time Series Data with xts and zoo in R
    		 Importing and Managing Financial Data in R
    		 Introduction to Portfolio Analysis in R
    		 Intermediate Portfolio Analysis in R
    
    	 Machine Learning Fundamentals, R
    		 Supervised Learning in R: Classification
    		 Supervised Learning in R: Regression
    		 Unsupervised Learning in R
    		 Machine Learning with caret in R
    		 Modeling with tidymodels in R
    		 Machine Learning with Tree-Based Models in R
    
    	 Text Mining, R
    		 Introduction to Text Analysis in R
    		 String Manipulation with stringr in R
    		 Text Mining with Bag-of-Words in R
    		 Sentiment Analysis in R
    
    	 Spatial Data, R
    		 Visualizing Geospatial Data in R
    		 Spatial Analysis with sf and raster in R
    		 Spatial Statistics in R
    		 Interactive Maps with leaflet in R
    
    	 Shiny Fundamentals, R
    		 Building Web Applications with Shiny in R
    		 Case Studies: Building Web Applications with Shiny in R
    		 Building Dashboards with shinydashboard
    		 Building Dashboards with flexdashboard
    
    	 Big Data, R
    		 Writing Efficient R Code
    		 Visualizing Big Data with Trelliscope in R
    		 Scalable Data Processing in R
    		 Introduction to Spark with sparklyr in R
    
    	 Tidyverse Fundamentals, R
    		 Introduction to the Tidyverse
    		 Reshaping Data with tidyr
    		 Dr. Semmelweis and the Discovery of Handwashing
    		 Modeling with Data in the Tidyverse
    		 Communicating with Data in the Tidyverse
    		 Categorical Data in the Tidyverse
    
    	 Network Analysis, R
    		 Network Analysis in R
    		 Predictive Analytics using Networked Data in R
    		 Network Analysis in the Tidyverse
    		 Case Studies: Network Analysis in R
    
    	 Supervised Machine Learning, R
    		 Machine Learning in the Tidyverse
    		 Intermediate Regression in R
    		 Modeling with tidymodels in R
    		 Machine Learning with Tree-Based Models in R
    		 Support Vector Machines in R
    		 Hyperparameter Tuning in R
    
    	 Interactive Data Visualization, R
    		 Interactive Maps with leaflet in R
    		 Interactive Data Visualization with plotly in R
    		 Intermediate Interactive Data Visualization with plotly in R
    		 Visualizing Big Data with Trelliscope in R
    
    	 Unsupervised Machine Learning, R
    		 Unsupervised Learning in R
    		 Cluster Analysis in R
    		 Factor Analysis in R
    
    	 Marketing Analytics, R
    		 Introduction to Text Analysis in R
    		 Analyzing Social Media Data in R
    		 Market Basket Analysis in R
    		 Machine Learning for Marketing Analytics in R
    		 Choice Modeling for Marketing in R
    		 Building Response Models in R
    
    	 Statistical Inference, R
    		 Foundations of Inference
    		 Inference for Categorical Data in R
    		 Inference for Numerical Data in R
    		 Inference for Linear Regression in R
    
    	 Intermediate Tidyverse Toolbox, R
    		 Dealing With Missing Data in R
    		 Foundations of Functional Programming with purrr
    		 Intermediate Functional Programming with purrr
    		 Machine Learning in the Tidyverse
    
    	 Analyzing Genomic Data, R
    		 Introduction to Bioconductor in R
    		 RNA-Seq with Bioconductor in R
    		 Differential Expression Analysis with limma in R
    		 ChIP-seq with Bioconductor in R
    
    	 Data Analyst, R
    		 Introduction to R
    		 Introduction to the Tidyverse
    		 Data Manipulation with dplyr 
    		 Joining Data with dplyr
    		 Introduction to Data Visualization with ggplot2
    		 Data Manipulation with R
    		 Exploratory Data Analysis in R
    		 Introduction to Statistics in R
    
    	 Data Scientist, R
    		 Introduction to R
    		 Intermediate R
    		 Introduction to the Tidyverse
    		 Data Manipulation with dplyr 
    		 Joining Data with dplyr
    		 Introduction to Data Visualization with ggplot2
    		 Intermediate Data Visualization with ggplot2
    		 Data Manipulation with R
    		 Reporting with R Markdown
    		 Introduction to Importing Data in R
    		 Intermediate Importing Data in R
    		 Cleaning Data in R
    		 Working with Dates and Times in R
    		 Importing & Cleaning Data with R
    		 Introduction to Writing Functions in R
    		 R Programming
    		 Exploratory Data Analysis in R
    		 Case Study: Exploratory Data Analysis in R
    		 Introduction to Statistics in R
    		 Introduction to Regression in R
    		 Intermediate Regression in R
    		 Supervised Learning in R: Classification
    
    	 R Programmer, R
    		 Introduction to the Tidyverse
    		 Dr. Semmelweis and the Discovery of Handwashing
    		 Data Manipulation with dplyr 
    		 Writing Efficient R Code
    		 Working with Dates and Times in R
    		 Drunken Datetimes in Ames, Iowa
    		 String Manipulation with stringr in R
    		 Data Manipulation with R
    		 Web Scraping in R
    		 Introduction to Writing Functions in R
    		 Clustering Bustabit Gambling Behavior
    		 Introduction to Shell
    		 Defensive R Programming
    		 Developing R Packages
    
    	 Quantitative Analyst, R
    		 Introduction to R for Finance
    		 Intermediate R for Finance
    		 Manipulating Time Series Data with xts and zoo in R
    		 Importing and Managing Financial Data in R
    		 Time Series Analysis in R
    		 ARIMA Models in R
    		 Case Studies: Manipulating Time Series Data in R
    		 Forecasting in R
    		 Visualizing Time Series Data in R
    		 Introduction to Portfolio Analysis in R
    		 Intermediate Portfolio Analysis in R
    		 Bond Valuation and Analysis in R
    		 Credit Risk Modeling in R
    		 Quantitative Risk Management in R
    		 Financial Trading in R
    
    	 Statistician, R
    		 Introduction to Statistics in R
    		 Foundations of Probability in R
    		 Introduction to Regression in R
    		 Intermediate Regression in R
    		 Sampling in R
    		 Hypothesis Testing in R
    		 Experimental Design in R
    		 Analyzing Survey Data in R
    		 Hierarchical and Mixed Effects Models in R
    		 Survival Analysis in R
    		 Fundamentals of Bayesian Data Analysis in R
    		 Factor Analysis in R
    		 Foundations of Inference
    
    	 Machine Learning Scientist, R
    		 Supervised Learning in R: Classification
    		 Supervised Learning in R: Regression
    		 Unsupervised Learning in R
    		 Machine Learning in the Tidyverse
    		 Intermediate Regression in R
    		 Cluster Analysis in R
    		 Machine Learning with caret in R
    		 Modeling with tidymodels in R
    		 Machine Learning with Tree-Based Models in R
    		 Machine Learning Fundamentals in R
    		 Support Vector Machines in R
    		 Fundamentals of Bayesian Data Analysis in R
    		 Hyperparameter Tuning in R
    		 Bayesian Regression Modeling with rstanarm
    
    
    Courses for Spreadsheet:
    	 Spreadsheet Fundamentals, Spreadsheet
    		 Data Analysis in Spreadsheets
    		 Intermediate Spreadsheets
    		 Pivot Tables in Spreadsheets
    		 Data Visualization in Spreadsheets
    
    	 Intermediate Spreadsheets, Spreadsheet
    		 Introduction to Statistics in Spreadsheets
    		 Error and Uncertainty in Spreadsheets
    		 Marketing Analytics in Spreadsheets
    
    	 Finance Fundamentals, Spreadsheet
    		 Financial Analytics in Spreadsheets
    		 Financial Modeling in Spreadsheets
    		 Loan Amortization in Spreadsheets
    
    
    Courses for SQL:
    	 SQL Fundamentals, SQL
    		 Introduction to SQL
    		 Joining Data in SQL
    		 Intermediate SQL
    		 PostgreSQL Summary Stats and Window Functions
    		 Functions for Manipulating Data in PostgreSQL
    
    	 SQL Server Fundamentals, SQL
    		 Introduction to SQL Server
    		 Joining Data in SQL
    		 Intermediate SQL Server
    		 Time Series Analysis in SQL Server
    		 Functions for Manipulating Data in SQL Server
    
    	 SQL for Business Analysts, SQL
    		 Exploratory Data Analysis in SQL
    		 Data-Driven Decision Making in SQL
    		 Applying SQL to Real-World Problems
    		 Analyzing Business Data in SQL
    		 Reporting in SQL
    
    	 SQL Server for Database Administrators, SQL
    		 Introduction to Relational Databases in SQL
    		 Database Design
    		 Transactions and Error Handling in SQL Server
    		 Writing Functions and Stored Procedures in SQL Server
    		 Building and Optimizing Triggers in SQL Server
    		 Improving Query Performance in SQL Server
    
    	 SQL for Database Administrators, SQL
    		 Introduction to Relational Databases in SQL
    		 Database Design
    		 Creating PostgreSQL Databases
    		 Improving Query Performance in PostgreSQL
    
    	 SQL Server Developer, SQL
    		 Introduction to SQL Server
    		 Introduction to Relational Databases in SQL
    		 Intermediate SQL Server
    		 Time Series Analysis in SQL Server
    		 Functions for Manipulating Data in SQL Server
    		 Database Design
    		 Transactions and Error Handling in SQL Server
    		 Writing Functions and Stored Procedures in SQL Server
    		 Building and Optimizing Triggers in SQL Server
    		 Improving Query Performance in SQL Server
    
    	 Data Analyst, SQL
    		 Data Visualization for Everyone
    		 Introduction to SQL
    		 Joining Data in SQL
    		 Intermediate SQL
    		 PostgreSQL Summary Stats and Window Functions
    		 Functions for Manipulating Data in PostgreSQL
    		 Exploratory Data Analysis in SQL
    		 Data-Driven Decision Making in SQL
    		 Data Communication Concepts
    
    

### Useful dictionaries

It is clear that this data is in a clear hierarchy, where tracks are arranged by technology, courses are sorted into tracks (sometimes in multiple tracks), and the courses themselves have different durations and descriptions.
It will be useful to split this data into several simple objects representing key-value relationships for later use.


```python
# # Get a dictionary of tracks and technologies, where the term after the comma is the technology
tracks_tech = dict()
for key in complete_tracks:
    # Get the technology
    tech = key.split(",")[1].strip()
    # Add the technology to the dictionary
    tracks_tech[key] = tech
tracks_tech
```




    {'R Programming, R': 'R',
     'Importing & Cleaning Data, R': 'R',
     'Data Visualization, R': 'R',
     'Data Manipulation, R': 'R',
     'Statistics Fundamentals, R': 'R',
     'Python Fundamentals, Python': 'Python',
     'Importing & Cleaning Data, Python': 'Python',
     'Data Manipulation, Python': 'Python',
     'Time Series, R': 'R',
     'Applied Finance, R': 'R',
     'Finance Fundamentals, R': 'R',
     'Machine Learning Fundamentals, R': 'R',
     'Machine Learning Fundamentals, Python': 'Python',
     'Text Mining, R': 'R',
     'Spatial Data, R': 'R',
     'Shiny Fundamentals, R': 'R',
     'Time Series, Python': 'Python',
     'Big Data, R': 'R',
     'Tidyverse Fundamentals, R': 'R',
     'Network Analysis, R': 'R',
     'Supervised Machine Learning, R': 'R',
     'Interactive Data Visualization, R': 'R',
     'SQL Fundamentals, SQL': 'SQL',
     'Statistics Fundamentals, Python': 'Python',
     'Unsupervised Machine Learning, R': 'R',
     'Marketing Analytics, R': 'R',
     'Statistical Inference, R': 'R',
     'Data Visualization, Python': 'Python',
     'Intermediate Tidyverse Toolbox, R': 'R',
     'SQL Server Fundamentals, SQL': 'SQL',
     'Image Processing, Python': 'Python',
     'Spreadsheet Fundamentals, Spreadsheet': 'Spreadsheet',
     'Python Toolbox, Python': 'Python',
     'Analyzing Genomic Data, R': 'R',
     'Big Data with PySpark, Python': 'Python',
     'Python Programming, Python': 'Python',
     'Data Skills for Business, Theory': 'Theory',
     'Marketing Analytics, Python': 'Python',
     'SQL for Business Analysts, SQL': 'SQL',
     'SQL Server for Database Administrators, SQL': 'SQL',
     'SQL for Database Administrators, SQL': 'SQL',
     'Intermediate Spreadsheets, Spreadsheet': 'Spreadsheet',
     'Deep Learning, Python': 'Python',
     'Data Literacy Fundamentals, Theory': 'Theory',
     'Natural Language Processing, Python': 'Python',
     'Deep Learning for NLP, Python': 'Python',
     'Finance Fundamentals, Python': 'Python',
     'Applied Finance, Python': 'Python',
     'Finance Fundamentals, Spreadsheet': 'Spreadsheet',
     'Tableau Fundamentals, Tableau': 'Tableau',
     'Power BI Fundamentals, Power BI': 'Power BI',
     'Data Analyst, R': 'R',
     'Data Scientist, R': 'R',
     'Data Analyst, Python': 'Python',
     'Data Scientist, Python': 'Python',
     'Python Programmer, Python': 'Python',
     'R Programmer, R': 'R',
     'Quantitative Analyst, R': 'R',
     'Statistician, R': 'R',
     'Machine Learning Scientist, Python': 'Python',
     'Data Engineer, Python': 'Python',
     'SQL Server Developer, SQL': 'SQL',
     'Data Analyst, Power BI': 'Power BI',
     'Data Analyst, SQL': 'SQL',
     'Machine Learning Scientist, R': 'R'}




```python
# For each course, get the number of tracks it is in
course_counts = dict()
for key in complete_tracks:
    for course in complete_tracks[key]:
        if course in course_counts:
            course_counts[course] += 1
        else:
            course_counts[course] = 1
# Sort the courses by the number of tracks they are in, and convert to a dictionary with the course as the key and the number of tracks as the value
course_counts = {x: course_counts[x] for x in sorted(course_counts, key = course_counts.get, reverse = True)}
course_counts
```




    {'Intermediate Regression in R': 5,
     'Data Manipulation with dplyr ': 4,
     'Introduction to Statistics in R': 4,
     'Data Manipulation with pandas': 4,
     'Introduction to the Tidyverse': 4,
     'Writing Efficient Python Code': 4,
     'Writing Functions in Python': 4,
     'Introduction to Relational Databases in SQL': 4,
     'Database Design': 4,
     'Introduction to R': 3,
     'Writing Efficient R Code': 3,
     'Introduction to Writing Functions in R': 3,
     'Introduction to Data Visualization with ggplot2': 3,
     'Joining Data with dplyr': 3,
     'Introduction to Regression in R': 3,
     'Intermediate Python': 3,
     'Python Data Science Toolbox (Part 1)': 3,
     'Python Data Science Toolbox (Part 2)': 3,
     'Joining Data with pandas': 3,
     'Manipulating Time Series Data with xts and zoo in R': 3,
     'Supervised Learning in R: Classification': 3,
     'Unsupervised Learning in R': 3,
     'Modeling with tidymodels in R': 3,
     'Machine Learning with Tree-Based Models in R': 3,
     'Supervised Learning with scikit-learn': 3,
     'Introduction to Deep Learning in Python': 3,
     'Dr. Semmelweis and the Discovery of Handwashing': 3,
     'Machine Learning in the Tidyverse': 3,
     'Joining Data in SQL': 3,
     'Introduction to Statistics in Python': 3,
     'Introduction to Data Visualization with Seaborn': 3,
     'Working with Dates and Times in Python': 3,
     'Introduction to PySpark': 3,
     'Unit Testing for Data Science in Python': 3,
     'Object-Oriented Programming in Python': 3,
     'Introduction to Data Science in Python': 3,
     'Data Manipulation with R': 3,
     'Introduction to Shell': 3,
     'Intermediate R': 2,
     'Introduction to Importing Data in R': 2,
     'Intermediate Importing Data in R': 2,
     'Cleaning Data in R': 2,
     'Reshaping Data with tidyr': 2,
     'Intermediate Data Visualization with ggplot2': 2,
     'Case Study: Exploratory Data Analysis in R': 2,
     'Sampling in R': 2,
     'Hypothesis Testing in R': 2,
     'Introduction to Python': 2,
     'Introduction to Importing Data in Python': 2,
     'Intermediate Importing Data in Python': 2,
     'Cleaning Data in Python': 2,
     'Analyzing Police Activity with pandas': 2,
     'Time Series Analysis in R': 2,
     'ARIMA Models in R': 2,
     'Forecasting in R': 2,
     'Visualizing Time Series Data in R': 2,
     'Case Studies: Manipulating Time Series Data in R': 2,
     'Quantitative Risk Management in R': 2,
     'Bond Valuation and Analysis in R': 2,
     'Financial Trading in R': 2,
     'Credit Risk Modeling in R': 2,
     'Introduction to R for Finance': 2,
     'Intermediate R for Finance': 2,
     'Importing and Managing Financial Data in R': 2,
     'Introduction to Portfolio Analysis in R': 2,
     'Intermediate Portfolio Analysis in R': 2,
     'Supervised Learning in R: Regression': 2,
     'Machine Learning with caret in R': 2,
     'Unsupervised Learning in Python': 2,
     'Linear Classifiers in Python': 2,
     'Introduction to Text Analysis in R': 2,
     'String Manipulation with stringr in R': 2,
     'Interactive Maps with leaflet in R': 2,
     'Manipulating Time Series Data in Python': 2,
     'Machine Learning for Time Series Data in Python': 2,
     'Visualizing Big Data with Trelliscope in R': 2,
     'Support Vector Machines in R': 2,
     'Hyperparameter Tuning in R': 2,
     'Introduction to SQL': 2,
     'Intermediate SQL': 2,
     'PostgreSQL Summary Stats and Window Functions': 2,
     'Functions for Manipulating Data in PostgreSQL': 2,
     'Introduction to Regression with statsmodels in Python': 2,
     'Sampling in Python': 2,
     'Hypothesis Testing in Python': 2,
     'Cluster Analysis in R': 2,
     'Factor Analysis in R': 2,
     'Foundations of Inference': 2,
     'Introduction to Data Visualization with Matplotlib': 2,
     'Introduction to SQL Server': 2,
     'Intermediate SQL Server': 2,
     'Time Series Analysis in SQL Server': 2,
     'Functions for Manipulating Data in SQL Server': 2,
     'Image Processing in Python': 2,
     'Image Processing with Keras in Python': 2,
     'Regular Expressions in Python': 2,
     'Big Data Fundamentals with PySpark': 2,
     'Machine Learning with PySpark': 2,
     'Software Engineering for Data Scientists in Python': 2,
     'Introduction to Statistics in Spreadsheets': 2,
     'Exploratory Data Analysis in SQL': 2,
     'Data-Driven Decision Making in SQL': 2,
     'Transactions and Error Handling in SQL Server': 2,
     'Writing Functions and Stored Procedures in SQL Server': 2,
     'Building and Optimizing Triggers in SQL Server': 2,
     'Improving Query Performance in SQL Server': 2,
     'Introduction to TensorFlow in Python': 2,
     'Introduction to Deep Learning with Keras': 2,
     'Advanced Deep Learning with Keras': 2,
     'Data Visualization for Everyone': 2,
     'Data Engineering for Everyone': 2,
     'Introduction to Natural Language Processing in Python': 2,
     'Feature Engineering for NLP in Python': 2,
     'Introduction to Power BI': 2,
     'Introduction to DAX in Power BI': 2,
     'Data Visualization in Power BI': 2,
     'Case Study: Analyzing Job Market Data in Power BI': 2,
     'Data Preparation in Power BI': 2,
     'Data Modeling in Power BI': 2,
     'Exploratory Data Analysis in R': 2,
     'Working with Dates and Times in R': 2,
     'Data Manipulation with Python': 2,
     'Exploratory Data Analysis in Python': 2,
     'Python Programming': 2,
     'Fundamentals of Bayesian Data Analysis in R': 2,
     'Object-Oriented Programming with S3 and R6 in R': 1,
     'Exploring the NYC Airbnb Market': 1,
     'Visualization Best Practices in R': 1,
     'Data Manipulation with data.table in R': 1,
     'Joining Data with data.table in R': 1,
     'Reshaping Data with pandas': 1,
     'Introduction to Databases in Python': 1,
     'Equity Valuation in R': 1,
     'Life Insurance Products Valuation in R': 1,
     'GARCH Models in R': 1,
     'Text Mining with Bag-of-Words in R': 1,
     'Sentiment Analysis in R': 1,
     'Visualizing Geospatial Data in R': 1,
     'Spatial Analysis with sf and raster in R': 1,
     'Spatial Statistics in R': 1,
     'Building Web Applications with Shiny in R': 1,
     'Case Studies: Building Web Applications with Shiny in R': 1,
     'Building Dashboards with shinydashboard': 1,
     'Building Dashboards with flexdashboard': 1,
     'Time Series Analysis in Python': 1,
     'Visualizing Time Series Data in Python': 1,
     'ARIMA Models in Python': 1,
     'Scalable Data Processing in R': 1,
     'Introduction to Spark with sparklyr in R': 1,
     'Modeling with Data in the Tidyverse': 1,
     'Communicating with Data in the Tidyverse': 1,
     'Categorical Data in the Tidyverse': 1,
     'Network Analysis in R': 1,
     'Predictive Analytics using Networked Data in R': 1,
     'Network Analysis in the Tidyverse': 1,
     'Case Studies: Network Analysis in R': 1,
     'Interactive Data Visualization with plotly in R': 1,
     'Intermediate Interactive Data Visualization with plotly in R': 1,
     'Intermediate Regression with statsmodels in Python': 1,
     'Analyzing Social Media Data in R': 1,
     'Market Basket Analysis in R': 1,
     'Machine Learning for Marketing Analytics in R': 1,
     'Choice Modeling for Marketing in R': 1,
     'Building Response Models in R': 1,
     'Inference for Categorical Data in R': 1,
     'Inference for Numerical Data in R': 1,
     'Inference for Linear Regression in R': 1,
     'Improving Your Data Visualizations in Python': 1,
     'Visualizing Geospatial Data in Python': 1,
     'Dealing With Missing Data in R': 1,
     'Foundations of Functional Programming with purrr': 1,
     'Intermediate Functional Programming with purrr': 1,
     'Biomedical Image Analysis in Python': 1,
     'Data Analysis in Spreadsheets': 1,
     'Intermediate Spreadsheets': 1,
     'Pivot Tables in Spreadsheets': 1,
     'Data Visualization in Spreadsheets': 1,
     'Dealing with Missing Data in Python': 1,
     'Introduction to Bioconductor in R': 1,
     'RNA-Seq with Bioconductor in R': 1,
     'Differential Expression Analysis with limma in R': 1,
     'ChIP-seq with Bioconductor in R': 1,
     'Cleaning Data with PySpark': 1,
     'Feature Engineering with PySpark': 1,
     'Building Recommendation Engines with PySpark': 1,
     'Writing Efficient Code with pandas': 1,
     'Data Science for Business': 1,
     'Machine Learning for Business': 1,
     'Data-Driven Decision Making for Business': 1,
     'Marketing Analytics for Business': 1,
     'AI Fundamentals': 1,
     'Analyzing Marketing Campaigns with pandas': 1,
     'Analyzing Social Media Data in Python': 1,
     'Market Basket Analysis in Python': 1,
     'Machine Learning for Marketing in Python': 1,
     'Customer Segmentation in Python': 1,
     'Marketing Analytics: Predicting Customer Churn in Python': 1,
     'Customer Analytics and A/B Testing in Python': 1,
     'Applying SQL to Real-World Problems': 1,
     'Analyzing Business Data in SQL': 1,
     'Reporting in SQL': 1,
     'Creating PostgreSQL Databases': 1,
     'Improving Query Performance in PostgreSQL': 1,
     'Error and Uncertainty in Spreadsheets': 1,
     'Marketing Analytics in Spreadsheets': 1,
     'Introduction to Deep Learning with PyTorch': 1,
     'Data Science for Everyone': 1,
     'Machine Learning for Everyone': 1,
     'Cloud Computing for Everyone': 1,
     'Sentiment Analysis in Python': 1,
     'Building Chatbots in Python': 1,
     'Advanced NLP with spaCy': 1,
     'Spoken Language Processing in Python': 1,
     'Recurrent Neural Networks for Language Modeling in Python': 1,
     'Machine Translation in Python': 1,
     'Natural Language Generation in Python': 1,
     'Introduction to Python for Finance': 1,
     'Intermediate Python for Finance': 1,
     'Introduction to Financial Concepts in Python': 1,
     'Importing and Managing Financial Data in Python': 1,
     'Introduction to Portfolio Analysis in Python': 1,
     'Introduction to Portfolio Risk Management in Python': 1,
     'Quantitative Risk Management in Python': 1,
     'Credit Risk Modeling in Python': 1,
     'GARCH Models in Python': 1,
     'Financial Analytics in Spreadsheets': 1,
     'Financial Modeling in Spreadsheets': 1,
     'Loan Amortization in Spreadsheets': 1,
     'Introduction to Tableau': 1,
     'Analyzing Data in Tableau': 1,
     'Creating Dashboards in Tableau': 1,
     'Case Study: Analyzing Customer Churn in Tableau': 1,
     'Connecting Data in Tableau': 1,
     'Reporting with R Markdown': 1,
     'Importing & Cleaning Data with R': 1,
     'R Programming': 1,
     'Investigating Netflix Movies and Guest Stars in The Office': 1,
     'The GitHub History of the Scala Language': 1,
     'Introduction to NumPy': 1,
     'The Android App Market on Google Play': 1,
     'Intermediate Data Visualization with Seaborn': 1,
     'A Visual History of Nobel Prize Winners': 1,
     'Importing & Cleaning Data with Python': 1,
     'Predicting Credit Card Approvals': 1,
     'Data Types for Data Science in Python': 1,
     'Web Scraping in Python': 1,
     'Developing Python Packages': 1,
     'Drunken Datetimes in Ames, Iowa': 1,
     'Web Scraping in R': 1,
     'Clustering Bustabit Gambling Behavior': 1,
     'Defensive R Programming': 1,
     'Developing R Packages': 1,
     'Foundations of Probability in R': 1,
     'Experimental Design in R': 1,
     'Analyzing Survey Data in R': 1,
     'Hierarchical and Mixed Effects Models in R': 1,
     'Survival Analysis in R': 1,
     'Machine Learning with Tree-Based Models in Python': 1,
     'Extreme Gradient Boosting with XGBoost': 1,
     'Cluster Analysis in Python': 1,
     'Dimensionality Reduction in Python': 1,
     'Preprocessing for Machine Learning in Python': 1,
     'Feature Engineering for Machine Learning in Python': 1,
     'Model Validation in Python': 1,
     'Machine Learning Fundamentals in Python': 1,
     'Hyperparameter Tuning in Python': 1,
     'Introduction to Data Engineering': 1,
     'Streamlined Data Ingestion with pandas': 1,
     'Data Processing in Shell': 1,
     'Introduction to Bash Scripting': 1,
     'Introduction to Airflow in Python': 1,
     'Introduction to AWS Boto in Python': 1,
     'Data Analysis in SQL (PostgreSQL)': 1,
     'Introduction to Scala': 1,
     'Case Study: Analyzing Customer Churn in Power BI': 1,
     'Data Transformation in Power BI': 1,
     'Intermediate Data Modeling in Power BI': 1,
     'DAX Functions in Power BI': 1,
     'Intermediate DAX in Power BI': 1,
     'User-Oriented Design in Power BI': 1,
     'Exploratory Data Analysis in Power BI': 1,
     'Trend Analysis in Power BI': 1,
     'Reports in Power BI': 1,
     'Report Design in Power BI': 1,
     'Data Connections in Power BI': 1,
     'Deploying and Maintaining Assets in Power BI': 1,
     'Data Communication Concepts': 1,
     'Machine Learning Fundamentals in R': 1,
     'Bayesian Regression Modeling with rstanarm': 1}




```python
# Create a dictionary of courses and the tracks they are in
course_tracks = dict()
for key in complete_tracks:
    for course in complete_tracks[key]:
        if course in course_tracks:
            course_tracks[course].append(key)
        else:
            course_tracks[course] = [key]
course_tracks
```




    {'Introduction to R': ['R Programming, R',
      'Data Analyst, R',
      'Data Scientist, R'],
     'Intermediate R': ['R Programming, R', 'Data Scientist, R'],
     'Writing Efficient R Code': ['R Programming, R',
      'Big Data, R',
      'R Programmer, R'],
     'Introduction to Writing Functions in R': ['R Programming, R',
      'Data Scientist, R',
      'R Programmer, R'],
     'Object-Oriented Programming with S3 and R6 in R': ['R Programming, R'],
     'Introduction to Importing Data in R': ['Importing & Cleaning Data, R',
      'Data Scientist, R'],
     'Intermediate Importing Data in R': ['Importing & Cleaning Data, R',
      'Data Scientist, R'],
     'Cleaning Data in R': ['Importing & Cleaning Data, R', 'Data Scientist, R'],
     'Reshaping Data with tidyr': ['Importing & Cleaning Data, R',
      'Tidyverse Fundamentals, R'],
     'Exploring the NYC Airbnb Market': ['Importing & Cleaning Data, R'],
     'Introduction to Data Visualization with ggplot2': ['Data Visualization, R',
      'Data Analyst, R',
      'Data Scientist, R'],
     'Intermediate Data Visualization with ggplot2': ['Data Visualization, R',
      'Data Scientist, R'],
     'Visualization Best Practices in R': ['Data Visualization, R'],
     'Data Manipulation with dplyr ': ['Data Manipulation, R',
      'Data Analyst, R',
      'Data Scientist, R',
      'R Programmer, R'],
     'Joining Data with dplyr': ['Data Manipulation, R',
      'Data Analyst, R',
      'Data Scientist, R'],
     'Case Study: Exploratory Data Analysis in R': ['Data Manipulation, R',
      'Data Scientist, R'],
     'Data Manipulation with data.table in R': ['Data Manipulation, R'],
     'Joining Data with data.table in R': ['Data Manipulation, R'],
     'Introduction to Statistics in R': ['Statistics Fundamentals, R',
      'Data Analyst, R',
      'Data Scientist, R',
      'Statistician, R'],
     'Introduction to Regression in R': ['Statistics Fundamentals, R',
      'Data Scientist, R',
      'Statistician, R'],
     'Intermediate Regression in R': ['Statistics Fundamentals, R',
      'Supervised Machine Learning, R',
      'Data Scientist, R',
      'Statistician, R',
      'Machine Learning Scientist, R'],
     'Sampling in R': ['Statistics Fundamentals, R', 'Statistician, R'],
     'Hypothesis Testing in R': ['Statistics Fundamentals, R', 'Statistician, R'],
     'Introduction to Python': ['Python Fundamentals, Python',
      'Data Scientist, Python'],
     'Intermediate Python': ['Python Fundamentals, Python',
      'Data Analyst, Python',
      'Data Scientist, Python'],
     'Python Data Science Toolbox (Part 1)': ['Python Fundamentals, Python',
      'Data Scientist, Python',
      'Python Programmer, Python'],
     'Python Data Science Toolbox (Part 2)': ['Python Fundamentals, Python',
      'Data Scientist, Python',
      'Python Programmer, Python'],
     'Introduction to Importing Data in Python': ['Importing & Cleaning Data, Python',
      'Data Scientist, Python'],
     'Intermediate Importing Data in Python': ['Importing & Cleaning Data, Python',
      'Data Scientist, Python'],
     'Cleaning Data in Python': ['Importing & Cleaning Data, Python',
      'Data Scientist, Python'],
     'Reshaping Data with pandas': ['Importing & Cleaning Data, Python'],
     'Data Manipulation with pandas': ['Data Manipulation, Python',
      'Data Analyst, Python',
      'Data Scientist, Python',
      'Python Programmer, Python'],
     'Joining Data with pandas': ['Data Manipulation, Python',
      'Data Analyst, Python',
      'Data Scientist, Python'],
     'Analyzing Police Activity with pandas': ['Data Manipulation, Python',
      'Data Scientist, Python'],
     'Introduction to Databases in Python': ['Data Manipulation, Python'],
     'Manipulating Time Series Data with xts and zoo in R': ['Time Series, R',
      'Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Time Series Analysis in R': ['Time Series, R', 'Quantitative Analyst, R'],
     'ARIMA Models in R': ['Time Series, R', 'Quantitative Analyst, R'],
     'Forecasting in R': ['Time Series, R', 'Quantitative Analyst, R'],
     'Visualizing Time Series Data in R': ['Time Series, R',
      'Quantitative Analyst, R'],
     'Case Studies: Manipulating Time Series Data in R': ['Time Series, R',
      'Quantitative Analyst, R'],
     'Quantitative Risk Management in R': ['Applied Finance, R',
      'Quantitative Analyst, R'],
     'Equity Valuation in R': ['Applied Finance, R'],
     'Life Insurance Products Valuation in R': ['Applied Finance, R'],
     'Bond Valuation and Analysis in R': ['Applied Finance, R',
      'Quantitative Analyst, R'],
     'Financial Trading in R': ['Applied Finance, R', 'Quantitative Analyst, R'],
     'Credit Risk Modeling in R': ['Applied Finance, R',
      'Quantitative Analyst, R'],
     'GARCH Models in R': ['Applied Finance, R'],
     'Introduction to R for Finance': ['Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Intermediate R for Finance': ['Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Importing and Managing Financial Data in R': ['Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Introduction to Portfolio Analysis in R': ['Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Intermediate Portfolio Analysis in R': ['Finance Fundamentals, R',
      'Quantitative Analyst, R'],
     'Supervised Learning in R: Classification': ['Machine Learning Fundamentals, R',
      'Data Scientist, R',
      'Machine Learning Scientist, R'],
     'Supervised Learning in R: Regression': ['Machine Learning Fundamentals, R',
      'Machine Learning Scientist, R'],
     'Unsupervised Learning in R': ['Machine Learning Fundamentals, R',
      'Unsupervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Machine Learning with caret in R': ['Machine Learning Fundamentals, R',
      'Machine Learning Scientist, R'],
     'Modeling with tidymodels in R': ['Machine Learning Fundamentals, R',
      'Supervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Machine Learning with Tree-Based Models in R': ['Machine Learning Fundamentals, R',
      'Supervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Supervised Learning with scikit-learn': ['Machine Learning Fundamentals, Python',
      'Data Scientist, Python',
      'Machine Learning Scientist, Python'],
     'Unsupervised Learning in Python': ['Machine Learning Fundamentals, Python',
      'Machine Learning Scientist, Python'],
     'Linear Classifiers in Python': ['Machine Learning Fundamentals, Python',
      'Machine Learning Scientist, Python'],
     'Introduction to Deep Learning in Python': ['Machine Learning Fundamentals, Python',
      'Deep Learning, Python',
      'Machine Learning Scientist, Python'],
     'Introduction to Text Analysis in R': ['Text Mining, R',
      'Marketing Analytics, R'],
     'String Manipulation with stringr in R': ['Text Mining, R',
      'R Programmer, R'],
     'Text Mining with Bag-of-Words in R': ['Text Mining, R'],
     'Sentiment Analysis in R': ['Text Mining, R'],
     'Visualizing Geospatial Data in R': ['Spatial Data, R'],
     'Spatial Analysis with sf and raster in R': ['Spatial Data, R'],
     'Spatial Statistics in R': ['Spatial Data, R'],
     'Interactive Maps with leaflet in R': ['Spatial Data, R',
      'Interactive Data Visualization, R'],
     'Building Web Applications with Shiny in R': ['Shiny Fundamentals, R'],
     'Case Studies: Building Web Applications with Shiny in R': ['Shiny Fundamentals, R'],
     'Building Dashboards with shinydashboard': ['Shiny Fundamentals, R'],
     'Building Dashboards with flexdashboard': ['Shiny Fundamentals, R'],
     'Manipulating Time Series Data in Python': ['Time Series, Python',
      'Finance Fundamentals, Python'],
     'Time Series Analysis in Python': ['Time Series, Python'],
     'Visualizing Time Series Data in Python': ['Time Series, Python'],
     'ARIMA Models in Python': ['Time Series, Python'],
     'Machine Learning for Time Series Data in Python': ['Time Series, Python',
      'Machine Learning Scientist, Python'],
     'Visualizing Big Data with Trelliscope in R': ['Big Data, R',
      'Interactive Data Visualization, R'],
     'Scalable Data Processing in R': ['Big Data, R'],
     'Introduction to Spark with sparklyr in R': ['Big Data, R'],
     'Introduction to the Tidyverse': ['Tidyverse Fundamentals, R',
      'Data Analyst, R',
      'Data Scientist, R',
      'R Programmer, R'],
     'Dr. Semmelweis and the Discovery of Handwashing': ['Tidyverse Fundamentals, R',
      'Data Scientist, Python',
      'R Programmer, R'],
     'Modeling with Data in the Tidyverse': ['Tidyverse Fundamentals, R'],
     'Communicating with Data in the Tidyverse': ['Tidyverse Fundamentals, R'],
     'Categorical Data in the Tidyverse': ['Tidyverse Fundamentals, R'],
     'Network Analysis in R': ['Network Analysis, R'],
     'Predictive Analytics using Networked Data in R': ['Network Analysis, R'],
     'Network Analysis in the Tidyverse': ['Network Analysis, R'],
     'Case Studies: Network Analysis in R': ['Network Analysis, R'],
     'Machine Learning in the Tidyverse': ['Supervised Machine Learning, R',
      'Intermediate Tidyverse Toolbox, R',
      'Machine Learning Scientist, R'],
     'Support Vector Machines in R': ['Supervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Hyperparameter Tuning in R': ['Supervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Interactive Data Visualization with plotly in R': ['Interactive Data Visualization, R'],
     'Intermediate Interactive Data Visualization with plotly in R': ['Interactive Data Visualization, R'],
     'Introduction to SQL': ['SQL Fundamentals, SQL', 'Data Analyst, SQL'],
     'Joining Data in SQL': ['SQL Fundamentals, SQL',
      'SQL Server Fundamentals, SQL',
      'Data Analyst, SQL'],
     'Intermediate SQL': ['SQL Fundamentals, SQL', 'Data Analyst, SQL'],
     'PostgreSQL Summary Stats and Window Functions': ['SQL Fundamentals, SQL',
      'Data Analyst, SQL'],
     'Functions for Manipulating Data in PostgreSQL': ['SQL Fundamentals, SQL',
      'Data Analyst, SQL'],
     'Introduction to Statistics in Python': ['Statistics Fundamentals, Python',
      'Data Analyst, Python',
      'Data Scientist, Python'],
     'Introduction to Regression with statsmodels in Python': ['Statistics Fundamentals, Python',
      'Data Scientist, Python'],
     'Intermediate Regression with statsmodels in Python': ['Statistics Fundamentals, Python'],
     'Sampling in Python': ['Statistics Fundamentals, Python',
      'Data Scientist, Python'],
     'Hypothesis Testing in Python': ['Statistics Fundamentals, Python',
      'Data Scientist, Python'],
     'Cluster Analysis in R': ['Unsupervised Machine Learning, R',
      'Machine Learning Scientist, R'],
     'Factor Analysis in R': ['Unsupervised Machine Learning, R',
      'Statistician, R'],
     'Analyzing Social Media Data in R': ['Marketing Analytics, R'],
     'Market Basket Analysis in R': ['Marketing Analytics, R'],
     'Machine Learning for Marketing Analytics in R': ['Marketing Analytics, R'],
     'Choice Modeling for Marketing in R': ['Marketing Analytics, R'],
     'Building Response Models in R': ['Marketing Analytics, R'],
     'Foundations of Inference': ['Statistical Inference, R', 'Statistician, R'],
     'Inference for Categorical Data in R': ['Statistical Inference, R'],
     'Inference for Numerical Data in R': ['Statistical Inference, R'],
     'Inference for Linear Regression in R': ['Statistical Inference, R'],
     'Introduction to Data Visualization with Matplotlib': ['Data Visualization, Python',
      'Data Scientist, Python'],
     'Introduction to Data Visualization with Seaborn': ['Data Visualization, Python',
      'Data Analyst, Python',
      'Data Scientist, Python'],
     'Improving Your Data Visualizations in Python': ['Data Visualization, Python'],
     'Visualizing Geospatial Data in Python': ['Data Visualization, Python'],
     'Dealing With Missing Data in R': ['Intermediate Tidyverse Toolbox, R'],
     'Foundations of Functional Programming with purrr': ['Intermediate Tidyverse Toolbox, R'],
     'Intermediate Functional Programming with purrr': ['Intermediate Tidyverse Toolbox, R'],
     'Introduction to SQL Server': ['SQL Server Fundamentals, SQL',
      'SQL Server Developer, SQL'],
     'Intermediate SQL Server': ['SQL Server Fundamentals, SQL',
      'SQL Server Developer, SQL'],
     'Time Series Analysis in SQL Server': ['SQL Server Fundamentals, SQL',
      'SQL Server Developer, SQL'],
     'Functions for Manipulating Data in SQL Server': ['SQL Server Fundamentals, SQL',
      'SQL Server Developer, SQL'],
     'Image Processing in Python': ['Image Processing, Python',
      'Machine Learning Scientist, Python'],
     'Biomedical Image Analysis in Python': ['Image Processing, Python'],
     'Image Processing with Keras in Python': ['Image Processing, Python',
      'Machine Learning Scientist, Python'],
     'Data Analysis in Spreadsheets': ['Spreadsheet Fundamentals, Spreadsheet'],
     'Intermediate Spreadsheets': ['Spreadsheet Fundamentals, Spreadsheet'],
     'Pivot Tables in Spreadsheets': ['Spreadsheet Fundamentals, Spreadsheet'],
     'Data Visualization in Spreadsheets': ['Spreadsheet Fundamentals, Spreadsheet'],
     'Dealing with Missing Data in Python': ['Python Toolbox, Python'],
     'Working with Dates and Times in Python': ['Python Toolbox, Python',
      'Data Scientist, Python',
      'Python Programmer, Python'],
     'Regular Expressions in Python': ['Python Toolbox, Python',
      'Python Programmer, Python'],
     'Writing Efficient Python Code': ['Python Toolbox, Python',
      'Python Programming, Python',
      'Python Programmer, Python',
      'Data Engineer, Python'],
     'Introduction to Bioconductor in R': ['Analyzing Genomic Data, R'],
     'RNA-Seq with Bioconductor in R': ['Analyzing Genomic Data, R'],
     'Differential Expression Analysis with limma in R': ['Analyzing Genomic Data, R'],
     'ChIP-seq with Bioconductor in R': ['Analyzing Genomic Data, R'],
     'Introduction to PySpark': ['Big Data with PySpark, Python',
      'Machine Learning Scientist, Python',
      'Data Engineer, Python'],
     'Big Data Fundamentals with PySpark': ['Big Data with PySpark, Python',
      'Data Engineer, Python'],
     'Cleaning Data with PySpark': ['Big Data with PySpark, Python'],
     'Feature Engineering with PySpark': ['Big Data with PySpark, Python'],
     'Machine Learning with PySpark': ['Big Data with PySpark, Python',
      'Machine Learning Scientist, Python'],
     'Building Recommendation Engines with PySpark': ['Big Data with PySpark, Python'],
     'Writing Efficient Code with pandas': ['Python Programming, Python'],
     'Writing Functions in Python': ['Python Programming, Python',
      'Data Scientist, Python',
      'Python Programmer, Python',
      'Data Engineer, Python'],
     'Software Engineering for Data Scientists in Python': ['Python Programming, Python',
      'Python Programmer, Python'],
     'Unit Testing for Data Science in Python': ['Python Programming, Python',
      'Python Programmer, Python',
      'Data Engineer, Python'],
     'Object-Oriented Programming in Python': ['Python Programming, Python',
      'Python Programmer, Python',
      'Data Engineer, Python'],
     'Data Science for Business': ['Data Skills for Business, Theory'],
     'Introduction to Statistics in Spreadsheets': ['Data Skills for Business, Theory',
      'Intermediate Spreadsheets, Spreadsheet'],
     'Machine Learning for Business': ['Data Skills for Business, Theory'],
     'Data-Driven Decision Making for Business': ['Data Skills for Business, Theory'],
     'Marketing Analytics for Business': ['Data Skills for Business, Theory'],
     'AI Fundamentals': ['Data Skills for Business, Theory'],
     'Introduction to Data Science in Python': ['Data Skills for Business, Theory',
      'Data Analyst, Python',
      'Python Programmer, Python'],
     'Analyzing Marketing Campaigns with pandas': ['Marketing Analytics, Python'],
     'Analyzing Social Media Data in Python': ['Marketing Analytics, Python'],
     'Market Basket Analysis in Python': ['Marketing Analytics, Python'],
     'Machine Learning for Marketing in Python': ['Marketing Analytics, Python'],
     'Customer Segmentation in Python': ['Marketing Analytics, Python'],
     'Marketing Analytics: Predicting Customer Churn in Python': ['Marketing Analytics, Python'],
     'Customer Analytics and A/B Testing in Python': ['Marketing Analytics, Python'],
     'Exploratory Data Analysis in SQL': ['SQL for Business Analysts, SQL',
      'Data Analyst, SQL'],
     'Data-Driven Decision Making in SQL': ['SQL for Business Analysts, SQL',
      'Data Analyst, SQL'],
     'Applying SQL to Real-World Problems': ['SQL for Business Analysts, SQL'],
     'Analyzing Business Data in SQL': ['SQL for Business Analysts, SQL'],
     'Reporting in SQL': ['SQL for Business Analysts, SQL'],
     'Introduction to Relational Databases in SQL': ['SQL Server for Database Administrators, SQL',
      'SQL for Database Administrators, SQL',
      'Data Engineer, Python',
      'SQL Server Developer, SQL'],
     'Database Design': ['SQL Server for Database Administrators, SQL',
      'SQL for Database Administrators, SQL',
      'Data Engineer, Python',
      'SQL Server Developer, SQL'],
     'Transactions and Error Handling in SQL Server': ['SQL Server for Database Administrators, SQL',
      'SQL Server Developer, SQL'],
     'Writing Functions and Stored Procedures in SQL Server': ['SQL Server for Database Administrators, SQL',
      'SQL Server Developer, SQL'],
     'Building and Optimizing Triggers in SQL Server': ['SQL Server for Database Administrators, SQL',
      'SQL Server Developer, SQL'],
     'Improving Query Performance in SQL Server': ['SQL Server for Database Administrators, SQL',
      'SQL Server Developer, SQL'],
     'Creating PostgreSQL Databases': ['SQL for Database Administrators, SQL'],
     'Improving Query Performance in PostgreSQL': ['SQL for Database Administrators, SQL'],
     'Error and Uncertainty in Spreadsheets': ['Intermediate Spreadsheets, Spreadsheet'],
     'Marketing Analytics in Spreadsheets': ['Intermediate Spreadsheets, Spreadsheet'],
     'Introduction to TensorFlow in Python': ['Deep Learning, Python',
      'Machine Learning Scientist, Python'],
     'Introduction to Deep Learning with PyTorch': ['Deep Learning, Python'],
     'Introduction to Deep Learning with Keras': ['Deep Learning, Python',
      'Machine Learning Scientist, Python'],
     'Advanced Deep Learning with Keras': ['Deep Learning, Python',
      'Machine Learning Scientist, Python'],
     'Data Science for Everyone': ['Data Literacy Fundamentals, Theory'],
     'Machine Learning for Everyone': ['Data Literacy Fundamentals, Theory'],
     'Data Visualization for Everyone': ['Data Literacy Fundamentals, Theory',
      'Data Analyst, SQL'],
     'Data Engineering for Everyone': ['Data Literacy Fundamentals, Theory',
      'Data Engineer, Python'],
     'Cloud Computing for Everyone': ['Data Literacy Fundamentals, Theory'],
     'Introduction to Natural Language Processing in Python': ['Natural Language Processing, Python',
      'Machine Learning Scientist, Python'],
     'Sentiment Analysis in Python': ['Natural Language Processing, Python'],
     'Building Chatbots in Python': ['Natural Language Processing, Python'],
     'Advanced NLP with spaCy': ['Natural Language Processing, Python'],
     'Spoken Language Processing in Python': ['Natural Language Processing, Python'],
     'Feature Engineering for NLP in Python': ['Natural Language Processing, Python',
      'Machine Learning Scientist, Python'],
     'Recurrent Neural Networks for Language Modeling in Python': ['Deep Learning for NLP, Python'],
     'Machine Translation in Python': ['Deep Learning for NLP, Python'],
     'Natural Language Generation in Python': ['Deep Learning for NLP, Python'],
     'Introduction to Python for Finance': ['Finance Fundamentals, Python'],
     'Intermediate Python for Finance': ['Finance Fundamentals, Python'],
     'Introduction to Financial Concepts in Python': ['Finance Fundamentals, Python'],
     'Importing and Managing Financial Data in Python': ['Finance Fundamentals, Python'],
     'Introduction to Portfolio Analysis in Python': ['Finance Fundamentals, Python'],
     'Introduction to Portfolio Risk Management in Python': ['Applied Finance, Python'],
     'Quantitative Risk Management in Python': ['Applied Finance, Python'],
     'Credit Risk Modeling in Python': ['Applied Finance, Python'],
     'GARCH Models in Python': ['Applied Finance, Python'],
     'Financial Analytics in Spreadsheets': ['Finance Fundamentals, Spreadsheet'],
     'Financial Modeling in Spreadsheets': ['Finance Fundamentals, Spreadsheet'],
     'Loan Amortization in Spreadsheets': ['Finance Fundamentals, Spreadsheet'],
     'Introduction to Tableau': ['Tableau Fundamentals, Tableau'],
     'Analyzing Data in Tableau': ['Tableau Fundamentals, Tableau'],
     'Creating Dashboards in Tableau': ['Tableau Fundamentals, Tableau'],
     'Case Study: Analyzing Customer Churn in Tableau': ['Tableau Fundamentals, Tableau'],
     'Connecting Data in Tableau': ['Tableau Fundamentals, Tableau'],
     'Introduction to Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Introduction to DAX in Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Data Visualization in Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Case Study: Analyzing Job Market Data in Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Data Preparation in Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Data Modeling in Power BI': ['Power BI Fundamentals, Power BI',
      'Data Analyst, Power BI'],
     'Data Manipulation with R': ['Data Analyst, R',
      'Data Scientist, R',
      'R Programmer, R'],
     'Exploratory Data Analysis in R': ['Data Analyst, R', 'Data Scientist, R'],
     'Reporting with R Markdown': ['Data Scientist, R'],
     'Working with Dates and Times in R': ['Data Scientist, R', 'R Programmer, R'],
     'Importing & Cleaning Data with R': ['Data Scientist, R'],
     'R Programming': ['Data Scientist, R'],
     'Data Manipulation with Python': ['Data Analyst, Python',
      'Data Scientist, Python'],
     'Exploratory Data Analysis in Python': ['Data Analyst, Python',
      'Data Scientist, Python'],
     'Investigating Netflix Movies and Guest Stars in The Office': ['Data Scientist, Python'],
     'The GitHub History of the Scala Language': ['Data Scientist, Python'],
     'Introduction to NumPy': ['Data Scientist, Python'],
     'The Android App Market on Google Play': ['Data Scientist, Python'],
     'Intermediate Data Visualization with Seaborn': ['Data Scientist, Python'],
     'A Visual History of Nobel Prize Winners': ['Data Scientist, Python'],
     'Importing & Cleaning Data with Python': ['Data Scientist, Python'],
     'Python Programming': ['Data Scientist, Python', 'Data Engineer, Python'],
     'Predicting Credit Card Approvals': ['Data Scientist, Python'],
     'Data Types for Data Science in Python': ['Python Programmer, Python'],
     'Web Scraping in Python': ['Python Programmer, Python'],
     'Introduction to Shell': ['Python Programmer, Python',
      'R Programmer, R',
      'Data Engineer, Python'],
     'Developing Python Packages': ['Python Programmer, Python'],
     'Drunken Datetimes in Ames, Iowa': ['R Programmer, R'],
     'Web Scraping in R': ['R Programmer, R'],
     'Clustering Bustabit Gambling Behavior': ['R Programmer, R'],
     'Defensive R Programming': ['R Programmer, R'],
     'Developing R Packages': ['R Programmer, R'],
     'Foundations of Probability in R': ['Statistician, R'],
     'Experimental Design in R': ['Statistician, R'],
     'Analyzing Survey Data in R': ['Statistician, R'],
     'Hierarchical and Mixed Effects Models in R': ['Statistician, R'],
     'Survival Analysis in R': ['Statistician, R'],
     'Fundamentals of Bayesian Data Analysis in R': ['Statistician, R',
      'Machine Learning Scientist, R'],
     'Machine Learning with Tree-Based Models in Python': ['Machine Learning Scientist, Python'],
     'Extreme Gradient Boosting with XGBoost': ['Machine Learning Scientist, Python'],
     'Cluster Analysis in Python': ['Machine Learning Scientist, Python'],
     'Dimensionality Reduction in Python': ['Machine Learning Scientist, Python'],
     'Preprocessing for Machine Learning in Python': ['Machine Learning Scientist, Python'],
     'Feature Engineering for Machine Learning in Python': ['Machine Learning Scientist, Python'],
     'Model Validation in Python': ['Machine Learning Scientist, Python'],
     'Machine Learning Fundamentals in Python': ['Machine Learning Scientist, Python'],
     'Hyperparameter Tuning in Python': ['Machine Learning Scientist, Python'],
     'Introduction to Data Engineering': ['Data Engineer, Python'],
     'Streamlined Data Ingestion with pandas': ['Data Engineer, Python'],
     'Data Processing in Shell': ['Data Engineer, Python'],
     'Introduction to Bash Scripting': ['Data Engineer, Python'],
     'Introduction to Airflow in Python': ['Data Engineer, Python'],
     'Introduction to AWS Boto in Python': ['Data Engineer, Python'],
     'Data Analysis in SQL (PostgreSQL)': ['Data Engineer, Python'],
     'Introduction to Scala': ['Data Engineer, Python'],
     'Case Study: Analyzing Customer Churn in Power BI': ['Data Analyst, Power BI'],
     'Data Transformation in Power BI': ['Data Analyst, Power BI'],
     'Intermediate Data Modeling in Power BI': ['Data Analyst, Power BI'],
     'DAX Functions in Power BI': ['Data Analyst, Power BI'],
     'Intermediate DAX in Power BI': ['Data Analyst, Power BI'],
     'User-Oriented Design in Power BI': ['Data Analyst, Power BI'],
     'Exploratory Data Analysis in Power BI': ['Data Analyst, Power BI'],
     'Trend Analysis in Power BI': ['Data Analyst, Power BI'],
     'Reports in Power BI': ['Data Analyst, Power BI'],
     'Report Design in Power BI': ['Data Analyst, Power BI'],
     'Data Connections in Power BI': ['Data Analyst, Power BI'],
     'Deploying and Maintaining Assets in Power BI': ['Data Analyst, Power BI'],
     'Data Communication Concepts': ['Data Analyst, SQL'],
     'Machine Learning Fundamentals in R': ['Machine Learning Scientist, R'],
     'Bayesian Regression Modeling with rstanarm': ['Machine Learning Scientist, R']}




```python
# Convert durations of each course to an integer
for key in complete_tracks:
    for course in complete_tracks[key]:
        course_info = complete_tracks[key][course]
        # Convert tuple to list
        course_info = list(course_info)
        # If the duration is a string, convert to an integer
        if isinstance(course_info[1], str):
            course_info[1] = int(course_info[1].split(" ")[0])
        # Update the dictionary
        complete_tracks[key][course] = course_info
complete_tracks

# Find the total duration of each track
track_durations = dict()
for key in complete_tracks:
    # Get the total duration of the track
    total_duration = sum([x[1] for x in complete_tracks[key].values()])
    # Add the total duration to the dictionary
    track_durations[key] = total_duration
# Sort the tracks by the total duration descending
track_durations = {x: track_durations[x] for x in sorted(track_durations, key = track_durations.get, reverse = False)}
track_durations
```




    {'Data Literacy Fundamentals, Theory': 10,
     'Data Visualization, R': 12,
     'Unsupervised Machine Learning, R': 12,
     'Image Processing, Python': 12,
     'Intermediate Spreadsheets, Spreadsheet': 12,
     'Deep Learning for NLP, Python': 12,
     'Finance Fundamentals, Spreadsheet': 12,
     'Importing & Cleaning Data, Python': 13,
     'Python Fundamentals, Python': 15,
     'Spreadsheet Fundamentals, Spreadsheet': 15,
     'Importing & Cleaning Data, R': 16,
     'Data Manipulation, Python': 16,
     'Machine Learning Fundamentals, Python': 16,
     'Text Mining, R': 16,
     'Spatial Data, R': 16,
     'Shiny Fundamentals, R': 16,
     'Big Data, R': 16,
     'Network Analysis, R': 16,
     'Interactive Data Visualization, R': 16,
     'Statistical Inference, R': 16,
     'Data Visualization, Python': 16,
     'Python Toolbox, Python': 16,
     'Analyzing Genomic Data, R': 16,
     'SQL for Database Administrators, SQL': 16,
     'Applied Finance, Python': 16,
     'Intermediate Tidyverse Toolbox, R': 17,
     'Power BI Fundamentals, Power BI': 17,
     'Data Manipulation, R': 20,
     'Statistics Fundamentals, R': 20,
     'Time Series, Python': 20,
     'Statistics Fundamentals, Python': 20,
     'Data Skills for Business, Theory': 20,
     'SQL for Business Analysts, SQL': 20,
     'Deep Learning, Python': 20,
     'SQL Fundamentals, SQL': 21,
     'R Programming, R': 22,
     'Tidyverse Fundamentals, R': 22,
     'SQL Server Fundamentals, SQL': 22,
     'Machine Learning Fundamentals, R': 24,
     'Marketing Analytics, R': 24,
     'Big Data with PySpark, Python': 24,
     'Python Programming, Python': 24,
     'SQL Server for Database Administrators, SQL': 24,
     'Time Series, R': 25,
     'Supervised Machine Learning, R': 25,
     'Natural Language Processing, Python': 25,
     'Finance Fundamentals, Python': 25,
     'Tableau Fundamentals, Tableau': 25,
     'Finance Fundamentals, R': 28,
     'Marketing Analytics, Python': 28,
     'Applied Finance, R': 30,
     'Data Analyst, R': 32,
     'Data Analyst, Python': 32,
     'Data Analyst, SQL': 34,
     'SQL Server Developer, SQL': 41,
     'R Programmer, R': 50,
     'Data Analyst, Power BI': 51,
     'Statistician, R': 52,
     'Machine Learning Scientist, R': 57,
     'Python Programmer, Python': 59,
     'Quantitative Analyst, R': 67,
     'Data Engineer, Python': 73,
     'Data Scientist, R': 88,
     'Machine Learning Scientist, Python': 93,
     'Data Scientist, Python': 109}



## Get current progress

It is key to have a list of completed courses, to allow the recommendations to adapt to my progress. I could create this manually, but I would prefer to scrape it to limit manual updating. Completed courses are indicated on the 'Courses' page when logged in to DataCamp. However, to avoid logging in to DataCamp through the `chromedriver`, which can get fiddly, I can instead access the Github repo where I store all of my DataCamp certificates by course name. This is doubly beneficial as it contains certificates from multiple DataCamp accounts I have held over time.


```python
# Get the completed courses as shown on Github
dc_done_url = "https://github.com/faisaljina/datacamp-profile/tree/main/Certificates"
dc_done_soup = get_html(dc_done_url)
dc_done_list = [x.text for x in dc_done_soup.find_all('span', class_ = 'css-truncate css-truncate-target d-block width-fit')]
# Remove the file extension from the file names
dc_done_list = [x.split(".")[0] for x in dc_done_list]
# Exclude the files that end in 'Track'
dc_done_list = [x for x in dc_done_list if not x.endswith("Track")]
dc_done_list
```




    ['ARIMAModelsinR',
     'CaseStudiesManipulatingTimeSeriesDatainR',
     'Data Manipulation with pandas',
     'DataManipulationwithdplyr',
     'DevelopingRPackages',
     'ExploratoryDataAnalysisinR',
     'ForecastinginR',
     'IntermediatePython',
     'IntermediateR',
     'IntermediateRforFinance',
     'IntroductiontoDataVisualizationwithggplot2',
     'IntroductiontoNaturalLanguageProcessinginPython',
     'IntroductiontoPython',
     'IntroductiontoR',
     'IntroductiontoRforFinance',
     'IntroductiontoSQL',
     'IntroductiontoSQLServer',
     'IntroductiontotheTidyverse',
     'JoiningDatainSQL',
     'ManipulatingTimeSeriesDatawithxtsandzooinR',
     'Python Data Science Toolbox (Part 2)',
     'PythonDataScienceToolbox(Part1)',
     'TimeSeriesAnalysisinR',
     'VisualizingTimeSeriesDatainR']




```python
# Get list of all possible DataCamp courses
all_courses = []
for key in complete_tracks:
    for course in complete_tracks[key]:
        all_courses.append(course)
all_courses = set(all_courses)
all_courses
```




    {'A Visual History of Nobel Prize Winners',
     'AI Fundamentals',
     'ARIMA Models in Python',
     'ARIMA Models in R',
     'Advanced Deep Learning with Keras',
     'Advanced NLP with spaCy',
     'Analyzing Business Data in SQL',
     'Analyzing Data in Tableau',
     'Analyzing Marketing Campaigns with pandas',
     'Analyzing Police Activity with pandas',
     'Analyzing Social Media Data in Python',
     'Analyzing Social Media Data in R',
     'Analyzing Survey Data in R',
     'Applying SQL to Real-World Problems',
     'Bayesian Regression Modeling with rstanarm',
     'Big Data Fundamentals with PySpark',
     'Biomedical Image Analysis in Python',
     'Bond Valuation and Analysis in R',
     'Building Chatbots in Python',
     'Building Dashboards with flexdashboard',
     'Building Dashboards with shinydashboard',
     'Building Recommendation Engines with PySpark',
     'Building Response Models in R',
     'Building Web Applications with Shiny in R',
     'Building and Optimizing Triggers in SQL Server',
     'Case Studies: Building Web Applications with Shiny in R',
     'Case Studies: Manipulating Time Series Data in R',
     'Case Studies: Network Analysis in R',
     'Case Study: Analyzing Customer Churn in Power BI',
     'Case Study: Analyzing Customer Churn in Tableau',
     'Case Study: Analyzing Job Market Data in Power BI',
     'Case Study: Exploratory Data Analysis in R',
     'Categorical Data in the Tidyverse',
     'ChIP-seq with Bioconductor in R',
     'Choice Modeling for Marketing in R',
     'Cleaning Data in Python',
     'Cleaning Data in R',
     'Cleaning Data with PySpark',
     'Cloud Computing for Everyone',
     'Cluster Analysis in Python',
     'Cluster Analysis in R',
     'Clustering Bustabit Gambling Behavior',
     'Communicating with Data in the Tidyverse',
     'Connecting Data in Tableau',
     'Creating Dashboards in Tableau',
     'Creating PostgreSQL Databases',
     'Credit Risk Modeling in Python',
     'Credit Risk Modeling in R',
     'Customer Analytics and A/B Testing in Python',
     'Customer Segmentation in Python',
     'DAX Functions in Power BI',
     'Data Analysis in SQL (PostgreSQL)',
     'Data Analysis in Spreadsheets',
     'Data Communication Concepts',
     'Data Connections in Power BI',
     'Data Engineering for Everyone',
     'Data Manipulation with Python',
     'Data Manipulation with R',
     'Data Manipulation with data.table in R',
     'Data Manipulation with dplyr ',
     'Data Manipulation with pandas',
     'Data Modeling in Power BI',
     'Data Preparation in Power BI',
     'Data Processing in Shell',
     'Data Science for Business',
     'Data Science for Everyone',
     'Data Transformation in Power BI',
     'Data Types for Data Science in Python',
     'Data Visualization for Everyone',
     'Data Visualization in Power BI',
     'Data Visualization in Spreadsheets',
     'Data-Driven Decision Making for Business',
     'Data-Driven Decision Making in SQL',
     'Database Design',
     'Dealing With Missing Data in R',
     'Dealing with Missing Data in Python',
     'Defensive R Programming',
     'Deploying and Maintaining Assets in Power BI',
     'Developing Python Packages',
     'Developing R Packages',
     'Differential Expression Analysis with limma in R',
     'Dimensionality Reduction in Python',
     'Dr. Semmelweis and the Discovery of Handwashing',
     'Drunken Datetimes in Ames, Iowa',
     'Equity Valuation in R',
     'Error and Uncertainty in Spreadsheets',
     'Experimental Design in R',
     'Exploratory Data Analysis in Power BI',
     'Exploratory Data Analysis in Python',
     'Exploratory Data Analysis in R',
     'Exploratory Data Analysis in SQL',
     'Exploring the NYC Airbnb Market',
     'Extreme Gradient Boosting with XGBoost',
     'Factor Analysis in R',
     'Feature Engineering for Machine Learning in Python',
     'Feature Engineering for NLP in Python',
     'Feature Engineering with PySpark',
     'Financial Analytics in Spreadsheets',
     'Financial Modeling in Spreadsheets',
     'Financial Trading in R',
     'Forecasting in R',
     'Foundations of Functional Programming with purrr',
     'Foundations of Inference',
     'Foundations of Probability in R',
     'Functions for Manipulating Data in PostgreSQL',
     'Functions for Manipulating Data in SQL Server',
     'Fundamentals of Bayesian Data Analysis in R',
     'GARCH Models in Python',
     'GARCH Models in R',
     'Hierarchical and Mixed Effects Models in R',
     'Hyperparameter Tuning in Python',
     'Hyperparameter Tuning in R',
     'Hypothesis Testing in Python',
     'Hypothesis Testing in R',
     'Image Processing in Python',
     'Image Processing with Keras in Python',
     'Importing & Cleaning Data with Python',
     'Importing & Cleaning Data with R',
     'Importing and Managing Financial Data in Python',
     'Importing and Managing Financial Data in R',
     'Improving Query Performance in PostgreSQL',
     'Improving Query Performance in SQL Server',
     'Improving Your Data Visualizations in Python',
     'Inference for Categorical Data in R',
     'Inference for Linear Regression in R',
     'Inference for Numerical Data in R',
     'Interactive Data Visualization with plotly in R',
     'Interactive Maps with leaflet in R',
     'Intermediate DAX in Power BI',
     'Intermediate Data Modeling in Power BI',
     'Intermediate Data Visualization with Seaborn',
     'Intermediate Data Visualization with ggplot2',
     'Intermediate Functional Programming with purrr',
     'Intermediate Importing Data in Python',
     'Intermediate Importing Data in R',
     'Intermediate Interactive Data Visualization with plotly in R',
     'Intermediate Portfolio Analysis in R',
     'Intermediate Python',
     'Intermediate Python for Finance',
     'Intermediate R',
     'Intermediate R for Finance',
     'Intermediate Regression in R',
     'Intermediate Regression with statsmodels in Python',
     'Intermediate SQL',
     'Intermediate SQL Server',
     'Intermediate Spreadsheets',
     'Introduction to AWS Boto in Python',
     'Introduction to Airflow in Python',
     'Introduction to Bash Scripting',
     'Introduction to Bioconductor in R',
     'Introduction to DAX in Power BI',
     'Introduction to Data Engineering',
     'Introduction to Data Science in Python',
     'Introduction to Data Visualization with Matplotlib',
     'Introduction to Data Visualization with Seaborn',
     'Introduction to Data Visualization with ggplot2',
     'Introduction to Databases in Python',
     'Introduction to Deep Learning in Python',
     'Introduction to Deep Learning with Keras',
     'Introduction to Deep Learning with PyTorch',
     'Introduction to Financial Concepts in Python',
     'Introduction to Importing Data in Python',
     'Introduction to Importing Data in R',
     'Introduction to Natural Language Processing in Python',
     'Introduction to NumPy',
     'Introduction to Portfolio Analysis in Python',
     'Introduction to Portfolio Analysis in R',
     'Introduction to Portfolio Risk Management in Python',
     'Introduction to Power BI',
     'Introduction to PySpark',
     'Introduction to Python',
     'Introduction to Python for Finance',
     'Introduction to R',
     'Introduction to R for Finance',
     'Introduction to Regression in R',
     'Introduction to Regression with statsmodels in Python',
     'Introduction to Relational Databases in SQL',
     'Introduction to SQL',
     'Introduction to SQL Server',
     'Introduction to Scala',
     'Introduction to Shell',
     'Introduction to Spark with sparklyr in R',
     'Introduction to Statistics in Python',
     'Introduction to Statistics in R',
     'Introduction to Statistics in Spreadsheets',
     'Introduction to Tableau',
     'Introduction to TensorFlow in Python',
     'Introduction to Text Analysis in R',
     'Introduction to Writing Functions in R',
     'Introduction to the Tidyverse',
     'Investigating Netflix Movies and Guest Stars in The Office',
     'Joining Data in SQL',
     'Joining Data with data.table in R',
     'Joining Data with dplyr',
     'Joining Data with pandas',
     'Life Insurance Products Valuation in R',
     'Linear Classifiers in Python',
     'Loan Amortization in Spreadsheets',
     'Machine Learning Fundamentals in Python',
     'Machine Learning Fundamentals in R',
     'Machine Learning for Business',
     'Machine Learning for Everyone',
     'Machine Learning for Marketing Analytics in R',
     'Machine Learning for Marketing in Python',
     'Machine Learning for Time Series Data in Python',
     'Machine Learning in the Tidyverse',
     'Machine Learning with PySpark',
     'Machine Learning with Tree-Based Models in Python',
     'Machine Learning with Tree-Based Models in R',
     'Machine Learning with caret in R',
     'Machine Translation in Python',
     'Manipulating Time Series Data in Python',
     'Manipulating Time Series Data with xts and zoo in R',
     'Market Basket Analysis in Python',
     'Market Basket Analysis in R',
     'Marketing Analytics for Business',
     'Marketing Analytics in Spreadsheets',
     'Marketing Analytics: Predicting Customer Churn in Python',
     'Model Validation in Python',
     'Modeling with Data in the Tidyverse',
     'Modeling with tidymodels in R',
     'Natural Language Generation in Python',
     'Network Analysis in R',
     'Network Analysis in the Tidyverse',
     'Object-Oriented Programming in Python',
     'Object-Oriented Programming with S3 and R6 in R',
     'Pivot Tables in Spreadsheets',
     'PostgreSQL Summary Stats and Window Functions',
     'Predicting Credit Card Approvals',
     'Predictive Analytics using Networked Data in R',
     'Preprocessing for Machine Learning in Python',
     'Python Data Science Toolbox (Part 1)',
     'Python Data Science Toolbox (Part 2)',
     'Python Programming',
     'Quantitative Risk Management in Python',
     'Quantitative Risk Management in R',
     'R Programming',
     'RNA-Seq with Bioconductor in R',
     'Recurrent Neural Networks for Language Modeling in Python',
     'Regular Expressions in Python',
     'Report Design in Power BI',
     'Reporting in SQL',
     'Reporting with R Markdown',
     'Reports in Power BI',
     'Reshaping Data with pandas',
     'Reshaping Data with tidyr',
     'Sampling in Python',
     'Sampling in R',
     'Scalable Data Processing in R',
     'Sentiment Analysis in Python',
     'Sentiment Analysis in R',
     'Software Engineering for Data Scientists in Python',
     'Spatial Analysis with sf and raster in R',
     'Spatial Statistics in R',
     'Spoken Language Processing in Python',
     'Streamlined Data Ingestion with pandas',
     'String Manipulation with stringr in R',
     'Supervised Learning in R: Classification',
     'Supervised Learning in R: Regression',
     'Supervised Learning with scikit-learn',
     'Support Vector Machines in R',
     'Survival Analysis in R',
     'Text Mining with Bag-of-Words in R',
     'The Android App Market on Google Play',
     'The GitHub History of the Scala Language',
     'Time Series Analysis in Python',
     'Time Series Analysis in R',
     'Time Series Analysis in SQL Server',
     'Transactions and Error Handling in SQL Server',
     'Trend Analysis in Power BI',
     'Unit Testing for Data Science in Python',
     'Unsupervised Learning in Python',
     'Unsupervised Learning in R',
     'User-Oriented Design in Power BI',
     'Visualization Best Practices in R',
     'Visualizing Big Data with Trelliscope in R',
     'Visualizing Geospatial Data in Python',
     'Visualizing Geospatial Data in R',
     'Visualizing Time Series Data in Python',
     'Visualizing Time Series Data in R',
     'Web Scraping in Python',
     'Web Scraping in R',
     'Working with Dates and Times in Python',
     'Working with Dates and Times in R',
     'Writing Efficient Code with pandas',
     'Writing Efficient Python Code',
     'Writing Efficient R Code',
     'Writing Functions and Stored Procedures in SQL Server',
     'Writing Functions in Python'}




```python
# For each course, check if it is in the scraped list of completed courses
completed_courses = []
for course in all_courses:
    # Remove colons from the course name
    course_cleaned = course.replace(":", "")
    # Split course name into list of words
    course_words = course_cleaned.split(" ")
    # Check if all words in the course name are in any of the dc_done_list
    for comp_course in dc_done_list:
        # Remove all spaces from comp_course if present
        comp_course = comp_course.replace(" ", "")
        # Remove the file extension from comp_course if present
        comp_course = comp_course.split(".")[0]
        # Check if all words in the course name are in the comp_course
        if all(word in comp_course for word in course_words):
            # Check if removing the matching words leaves no characters - this filters out similar courses with matching stems
            if len(comp_course.replace("".join(course_words), "")) == 0:
                completed_courses.append(course)
                break
# Sort alphabetically
completed_courses = sorted(completed_courses)
# Check the list is complete
if len(completed_courses) == len(dc_done_list):
    print("List is complete")
    print(f'Completed: {len(completed_courses)}')
else:
    print("List is not complete")
    # Show scraped and completed courses lengths
    print(f'Scraped: {len(dc_done_list)}')
    print(f'Matched: {len(completed_courses)}')
completed_courses
```

    List is complete
    Completed: 24
    




    ['ARIMA Models in R',
     'Case Studies: Manipulating Time Series Data in R',
     'Data Manipulation with dplyr ',
     'Data Manipulation with pandas',
     'Developing R Packages',
     'Exploratory Data Analysis in R',
     'Forecasting in R',
     'Intermediate Python',
     'Intermediate R',
     'Intermediate R for Finance',
     'Introduction to Data Visualization with ggplot2',
     'Introduction to Natural Language Processing in Python',
     'Introduction to Python',
     'Introduction to R',
     'Introduction to R for Finance',
     'Introduction to SQL',
     'Introduction to SQL Server',
     'Introduction to the Tidyverse',
     'Joining Data in SQL',
     'Manipulating Time Series Data with xts and zoo in R',
     'Python Data Science Toolbox (Part 1)',
     'Python Data Science Toolbox (Part 2)',
     'Time Series Analysis in R',
     'Visualizing Time Series Data in R']




```python
# Get dictionary of all tracks of courses minus the completed courses
tracks_minus_completed = dict()
for key in complete_tracks:
    # Get the list of courses for this track
    courses = complete_tracks[key]
    # Remove the completed courses from the list
    courses = {x: courses[x] for x in courses if x not in completed_courses}
    # Add the courses to the dictionary
    tracks_minus_completed[key] = courses
tracks_minus_completed['Python Fundamentals, Python']

# Get dictionary of track remaining durations
track_remaining_durations = dict()
for key in tracks_minus_completed:
    # Get the total duration of the track
    total_duration = sum([x[1] for x in tracks_minus_completed[key].values()])
    # Add the total duration to the dictionary
    track_remaining_durations[key] = total_duration
# Sort the tracks by the total duration descending
track_remaining_durations = {x: track_remaining_durations[x] for x in sorted(track_remaining_durations, key = track_remaining_durations.get, reverse = False)}
# If the total duration is 0, remove the track from durations and tracks_minus_completed
for key in track_remaining_durations.copy():
    if track_remaining_durations[key] == 0:
        del track_remaining_durations[key]
        del tracks_minus_completed[key]
track_remaining_durations

```




    {'Data Visualization, R': 8,
     'Data Literacy Fundamentals, Theory': 10,
     'R Programming, R': 12,
     'Data Manipulation, Python': 12,
     'SQL Fundamentals, SQL': 12,
     'Unsupervised Machine Learning, R': 12,
     'Image Processing, Python': 12,
     'Intermediate Spreadsheets, Spreadsheet': 12,
     'Deep Learning for NLP, Python': 12,
     'Finance Fundamentals, Spreadsheet': 12,
     'Data Analyst, R': 12,
     'Importing & Cleaning Data, Python': 13,
     'SQL Server Fundamentals, SQL': 13,
     'Finance Fundamentals, R': 15,
     'Spreadsheet Fundamentals, Spreadsheet': 15,
     'Importing & Cleaning Data, R': 16,
     'Data Manipulation, R': 16,
     'Machine Learning Fundamentals, Python': 16,
     'Text Mining, R': 16,
     'Spatial Data, R': 16,
     'Shiny Fundamentals, R': 16,
     'Big Data, R': 16,
     'Network Analysis, R': 16,
     'Interactive Data Visualization, R': 16,
     'Statistical Inference, R': 16,
     'Data Visualization, Python': 16,
     'Python Toolbox, Python': 16,
     'Analyzing Genomic Data, R': 16,
     'SQL for Database Administrators, SQL': 16,
     'Applied Finance, Python': 16,
     'Intermediate Tidyverse Toolbox, R': 17,
     'Power BI Fundamentals, Power BI': 17,
     'Tidyverse Fundamentals, R': 18,
     'Statistics Fundamentals, R': 20,
     'Time Series, Python': 20,
     'Statistics Fundamentals, Python': 20,
     'Data Skills for Business, Theory': 20,
     'SQL for Business Analysts, SQL': 20,
     'Deep Learning, Python': 20,
     'Natural Language Processing, Python': 21,
     'Machine Learning Fundamentals, R': 24,
     'Marketing Analytics, R': 24,
     'Big Data with PySpark, Python': 24,
     'Python Programming, Python': 24,
     'SQL Server for Database Administrators, SQL': 24,
     'Data Analyst, Python': 24,
     'Supervised Machine Learning, R': 25,
     'Finance Fundamentals, Python': 25,
     'Tableau Fundamentals, Tableau': 25,
     'Data Analyst, SQL': 25,
     'Marketing Analytics, Python': 28,
     'Applied Finance, R': 30,
     'Quantitative Analyst, R': 33,
     'SQL Server Developer, SQL': 37,
     'R Programmer, R': 38,
     'Python Programmer, Python': 48,
     'Data Analyst, Power BI': 51,
     'Statistician, R': 52,
     'Machine Learning Scientist, R': 57,
     'Data Scientist, R': 62,
     'Data Engineer, Python': 73,
     'Machine Learning Scientist, Python': 89,
     'Data Scientist, Python': 90}



## The Recommender is born

We have all the data in all the forms we need. Now we just need to write some appropriate functions to get us the recommender we have always wanted.


```python
# Function taking the technology name, and returning the track names and remaining durations
def get_tracks_by_tech(tech, k_tracks=3):
    # Get the list of tracks for this technology, where the tracks are the keys in the dictionary
    tracks = [x for x in tracks_minus_completed if tech in x]
    # Get the remaining durations for the tracks
    durations = [track_remaining_durations[x] for x in tracks]
    # Zip the tracks and durations together
    tracks_durations = list(zip(tracks, durations))
    # Sort the tracks by the remaining duration
    tracks_durations = sorted(tracks_durations, key = lambda x: x[1])
    # Return the k-smallest remaining durations and the tracks
    return tracks_durations[:k_tracks]

# Function taking the technology, finding the tracks and remaining durations and returning the courses within those tracks
def get_courses_by_tech(tech, k_tracks=3)->list:
    """
    Get the courses for a technology
    
    Parameters
    ----------
    tech : str
        The technology name
        k_tracks : int, optional
            The number of tracks to return courses from. The default is 3.
            
            Returns
            -------
            list
                The list of courses as a list of tuples of the form (course, duration)
                
                Examples
                --------
                get_courses_by_tech("Python")
                get_courses_by_tech("Python", k_tracks=2)
    """
    # Get the tracks and remaining durations for this technology
    tracks_durations = get_tracks_by_tech(tech, k_tracks)
    # Get the courses and durations for the selected tracks
    courses_durations = []
    for track, duration in tracks_durations:
        # Get the courses for this track
        course = tracks_minus_completed[track]
        # Get the remaining durations for the course
        duration = [course[x][1] for x in course]
        # Zip the course and duration together
        course_duration = list(zip(course, duration))
        # Add the course and duration to the list
        courses_durations.append(course_duration)
    # Flatten the list of lists of courses and durations
    courses_durations = [item for sublist in courses_durations for item in sublist]
    return courses_durations

get_courses_by_tech('Python', k_tracks=5)
```




    [('Joining Data with pandas', 4),
     ('Analyzing Police Activity with pandas', 4),
     ('Introduction to Databases in Python', 4),
     ('Image Processing in Python', 4),
     ('Biomedical Image Analysis in Python', 4),
     ('Image Processing with Keras in Python', 4),
     ('Recurrent Neural Networks for Language Modeling in Python', 4),
     ('Machine Translation in Python', 4),
     ('Natural Language Generation in Python', 4),
     ('Introduction to Importing Data in Python', 3),
     ('Intermediate Importing Data in Python', 2),
     ('Cleaning Data in Python', 4),
     ('Reshaping Data with pandas', 4),
     ('Supervised Learning with scikit-learn', 4),
     ('Unsupervised Learning in Python', 4),
     ('Linear Classifiers in Python', 4),
     ('Introduction to Deep Learning in Python', 4)]



So we have a function that find the course names and durations for the k-shortest remaining courses for any given technology.
We want to do something similar for a given course name, so that we can find the single shortest track doing that course would contribute towards.


```python
# Function taking a course name and finding which tracks it is in, and the remaining duration of the shortest remaining track
def get_track_by_course(course) -> Tuple[str, int]:
    """
    Get the track name and remaining duration of the shortest remaining track for a course

    Parameters
    ----------
    course : str
        The course name

    Returns
    -------
    track : str
        The track name
    duration : int
        The remaining duration of the shortest remaining track
    """
    # Get the list of tracks for this course
    tracks = course_tracks[course]
    # Get the remaining durations for the tracks
    durations = [track_remaining_durations[x] for x in tracks]
    # Zip the tracks and durations together
    tracks_durations = list(zip(tracks, durations))
    # If there is only one track, return the track and remaining duration
    if len(tracks_durations) == 1:
        return tracks_durations[0]
    # Sort the tracks by the remaining duration
    tracks_durations = sorted(tracks_durations, key = lambda x: x[1])
    # Check if the remaining duration of the two shortest remaining tracks are the same
    if tracks_durations[0][1] == tracks_durations[1][1]:
        # If so, return both tracks
        return tracks_durations[0], tracks_durations[1]
    else:
        # If not, return the shortest remaining track
        return tracks_durations[0]

get_track_by_course('Introduction to Statistics in Spreadsheets')
```




    ('Intermediate Spreadsheets, Spreadsheet', 12)



Last couple of things now, just want to double-check I can find the number of tracks a course appears in.


```python
# Check no. of tracks a course is in
test_course = 'Joining Data with pandas'
course_counts[test_course]
```




    3



Great, and just as some added sugar, I'd like to have a dictionary of courses and their descriptions.


```python
# Create a dictionary of all courses and their descriptions
course_descriptions = dict()
for track in complete_tracks:
    for course, description in complete_tracks[track].items():
        course_descriptions[course] = description[0]

# Print first few courses and their descriptions
for i, course in enumerate(course_descriptions.items()):
    if i == 4: break
    print(f'{course[0]}: {course[1]}')
```

    Introduction to R: Master the basics of data analysis by manipulating common data structures such as vectors, matrices, and data frames.
    Intermediate R: Continue your journey to becoming an R ninja by learning about conditional statements, loops, and vector functions.
    Writing Efficient R Code: Learn to write faster R code, discover benchmarking and profiling, and unlock the secrets of parallel programming.
    Introduction to Writing Functions in R: Use cluster analysis to glean insights into cryptocurrency gambling behavior.
    

### Put it all together

A dataframe is a suitable output for my recommendations as they display nicely in VSCode. Let's put all the above functions together and see what we get.


```python
# Function taking a technology and performing the above functions to get the courses, tracks and remaining durations
def get_recommendations(tech = None, k_tracks = 5, row_limit = 20)-> pd.DataFrame:
    """
    Take a technology and get recommended courses and info

    Args:
        tech (str): Technology name
        k_tracks (int): Number of tracks to return

    Returns:
        pd.DataFrame: Dataframe of recommended courses and associated information
    """

    if not tech:
        print(f"Choose a technology from:\n{[x for x in technologies]}\n(Defaulting to 'Python')")
        tech = "Python"
    # Get the courses for this technology
    courses = get_courses_by_tech(tech, k_tracks)
    # Get the tracks and track durations for each course
    tracks_durations = []
    for course in courses:
        track = get_track_by_course(course[0])
        tracks_durations.append(track)
    # Find the number of tracks for each course
    track_course_counts = []
    for course in courses:
        if course[0] in course_counts:
            track_course_counts.append(course_counts[course[0]])
        else:
            print("No count of course in tracks: ", course[0])
            track_course_counts.append(0)
    # Zip the courses, track durations data and track course counts together
    courses_all_info = list(zip(courses, tracks_durations, track_course_counts))
    # Flatten the course info
    courses_all_info = [item for sublist in courses_all_info for item in sublist]
    # Reshape the course info
    course_all_info_reshaped = []
    for i in range(len(courses_all_info)):
        if i % 3 == 0:
            course_all_info_reshaped.append(list(courses_all_info[i]) + list(courses_all_info[i+1]) + [courses_all_info[i+2]])
    # Put the course info into a dataframe
    course_all_info_df = pd.DataFrame(course_all_info_reshaped, columns = ['Course', 'Course Length', 'Shortest Track', 'Track Time Remaining', 'Track Duplication'])
    # Remove any duplicated lines
    course_all_info_df = course_all_info_df.drop_duplicates(subset = ['Course'])
    # Sort the dataframe by the track duration remaining ascending, then by the course Length ascending, then by the duplication in tracks descending, then by shortest track ascending
    course_all_info_df = course_all_info_df.sort_values(by = ['Track Time Remaining', 'Course Length', 'Track Duplication', 'Shortest Track'], ascending = [True, True, False, True])
    # Rearrange the columns to course, course length, duplication in tracks, track, track duration remaining
    course_all_info_df = course_all_info_df[['Course', 'Course Length', 'Track Duplication', 'Shortest Track', 'Track Time Remaining']]
    # Add a column for the course description
    course_all_info_df['Course Description'] = course_all_info_df['Course'].map(course_descriptions)
    # Reset the index
    course_all_info_df = course_all_info_df.reset_index(drop = True)
    # Limit the number of rows to the row_limit
    course_all_info_df = course_all_info_df.head(row_limit)
    # Return the dataframe
    return course_all_info_df
    

get_recommendations()

```

    Choose a technology from:
    ['Theory', 'Python', 'Tableau', 'Power BI', 'R', 'Spreadsheet', 'SQL']
    (Defaulting to 'Python')
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Joining Data with pandas</td>
      <td>4</td>
      <td>3</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>Learn to combine data from multiple tables by joining data together using pandas.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Analyzing Police Activity with pandas</td>
      <td>4</td>
      <td>2</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>Learn to draw conclusions from limited data using Python and statistics. This course covers everything from random sampling to stratified and cluster sampling.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Image Processing in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn powerful techniques for image analysis in Python using deep learning and convolutional neural networks in Keras.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Image Processing with Keras in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn to tune hyperparameters in Python.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Introduction to Databases in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>In this course, you'll learn the basics of relational databases and how to interact with them.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Recurrent Neural Networks for Language Modeling in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Use RNNs to classify text sentiment, generate sentences, and translate text between languages.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Machine Translation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Are you curious about the inner workings of the models that are behind products like Google Translate?</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Natural Language Generation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Imitate Shakespear, translate language and autocomplete sentences using Deep Learning in Python.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Biomedical Image Analysis in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn the fundamentals of exploring, manipulating, and measuring biomedical image data.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Intermediate Importing Data in Python</td>
      <td>2</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Learn to diagnose and treat dirty data and develop the skills needed to transform your raw data into accurate insights!</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Introduction to Importing Data in Python</td>
      <td>3</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Improve your Python data importing skills and learn to work with web and API data.</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Cleaning Data in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Learn how to work with dates and times in Python.</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Reshaping Data with pandas</td>
      <td>4</td>
      <td>1</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Reshape DataFrames from a wide to long format, stack and unstack rows and columns, and wrangle multi-index DataFrames.</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Supervised Learning with scikit-learn</td>
      <td>4</td>
      <td>3</td>
      <td>Machine Learning Fundamentals, Python</td>
      <td>16</td>
      <td>Grow your machine learning skills with scikit-learn in Python. Use real-world datasets in this interactive course and learn how to make powerful predictions!</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Introduction to Deep Learning in Python</td>
      <td>4</td>
      <td>3</td>
      <td>Machine Learning Fundamentals, Python</td>
      <td>16</td>
      <td>Learn to start developing deep learning models with Keras.</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Unsupervised Learning in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Machine Learning Fundamentals, Python</td>
      <td>16</td>
      <td>Learn how to cluster, transform, visualize, and extract insights from unlabeled datasets using scikit-learn and scipy.</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Linear Classifiers in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Machine Learning Fundamentals, Python</td>
      <td>16</td>
      <td>In this course you will learn the details of linear classifiers like logistic regression and SVM.</td>
    </tr>
  </tbody>
</table>
</div>



This has worked excellently. The key to this was the sorting of the data to suit my preference, and I think this ordering expresses that well.

## The Recommender System

Run this to get your recommendations!


```python
# Increase limit of string output in the dataframe to get the full course description
pd.set_option('display.max_colwidth', None)
```


```python
# Reorder technologies to preferred order
tech_list_ordered = ['Python', 'Power BI', 'SQL', 'Spreadsheet', 'Theory', 'Tableau', 'R']
# Add any extracted technologies not in the ordered list to the end of the list
tech_list_ordered += [x for x in technologies if x not in tech_list_ordered]

# Get recommendations for each technology
for tech in tech_list_ordered:
    print(f"{tech}:")
    display(get_recommendations(tech, k_tracks=4, row_limit=12))
    print("\n")
```

    Python:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Joining Data with pandas</td>
      <td>4</td>
      <td>3</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>Learn to combine data from multiple tables by joining data together using pandas.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Analyzing Police Activity with pandas</td>
      <td>4</td>
      <td>2</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>Learn to draw conclusions from limited data using Python and statistics. This course covers everything from random sampling to stratified and cluster sampling.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Image Processing in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn powerful techniques for image analysis in Python using deep learning and convolutional neural networks in Keras.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Image Processing with Keras in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn to tune hyperparameters in Python.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Introduction to Databases in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>In this course, you'll learn the basics of relational databases and how to interact with them.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Recurrent Neural Networks for Language Modeling in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Use RNNs to classify text sentiment, generate sentences, and translate text between languages.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Machine Translation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Are you curious about the inner workings of the models that are behind products like Google Translate?</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Natural Language Generation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Imitate Shakespear, translate language and autocomplete sentences using Deep Learning in Python.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Biomedical Image Analysis in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn the fundamentals of exploring, manipulating, and measuring biomedical image data.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Intermediate Importing Data in Python</td>
      <td>2</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Learn to diagnose and treat dirty data and develop the skills needed to transform your raw data into accurate insights!</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Introduction to Importing Data in Python</td>
      <td>3</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Improve your Python data importing skills and learn to work with web and API data.</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Cleaning Data in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Importing &amp; Cleaning Data, Python</td>
      <td>13</td>
      <td>Learn how to work with dates and times in Python.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    Power BI:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Introduction to DAX in Power BI</td>
      <td>2</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>Enhance your Power BI knowledge, by learning the fundamentals of Data Analysis Expressions (DAX) such as calculated columns, tables, and measures.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Introduction to Power BI</td>
      <td>3</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>Gain a 360° overview of how to explore and use Power BI to build impactful reports.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Data Visualization in Power BI</td>
      <td>3</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>Power BI is a powerful data visualization tool that can be used in reports and dashboards.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Case Study: Analyzing Job Market Data in Power BI</td>
      <td>3</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>Help a fictional company in this interactive Power BI case study. You’ll use Power Query, DAX, and dashboards to identify the most in-demand data jobs!</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Data Preparation in Power BI</td>
      <td>3</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>In this interactive Power BI course, you’ll learn how to use Power Query Editor to transform and shape your data to be ready for analysis.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Data Modeling in Power BI</td>
      <td>3</td>
      <td>2</td>
      <td>Power BI Fundamentals, Power BI</td>
      <td>17</td>
      <td>Learn the key concepts of data modeling on Power BI.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Data Connections in Power BI</td>
      <td>2</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>Discover the different ways you can enhance your Power BI data importing skills.</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Deploying and Maintaining Assets in Power BI</td>
      <td>2</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>Learn how to deploy and maintain assets in Power BI. You’ll get to grips with the Power BI Service interface and key elements in it like workspaces.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Case Study: Analyzing Customer Churn in Power BI</td>
      <td>3</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>You will investigate a dataset from a fictitious company called Databel in Power BI, and need to figure out why customers are churning.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Data Transformation in Power BI</td>
      <td>3</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>You’ll learn how to (un)pivot, transpose, append and join tables. Gain power with custom columns, M language, and the Advanced Editor.</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Intermediate Data Modeling in Power BI</td>
      <td>3</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>Master data modeling in Power BI.</td>
    </tr>
    <tr>
      <th>11</th>
      <td>DAX Functions in Power BI</td>
      <td>3</td>
      <td>1</td>
      <td>Data Analyst, Power BI</td>
      <td>51</td>
      <td>Data Analysis Expressions (DAX) allow you to take your Power BI skills to the next level by writing custom functions.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    SQL:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Intermediate SQL</td>
      <td>4</td>
      <td>2</td>
      <td>SQL Fundamentals, SQL</td>
      <td>12</td>
      <td>Master the complex SQL queries necessary to answer a wide variety of data science questions and prepare robust data sets for analysis in PostgreSQL.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PostgreSQL Summary Stats and Window Functions</td>
      <td>4</td>
      <td>2</td>
      <td>SQL Fundamentals, SQL</td>
      <td>12</td>
      <td>Learn how to create queries for analytics and data engineering with window functions, the SQL secret weapon!</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Functions for Manipulating Data in PostgreSQL</td>
      <td>4</td>
      <td>2</td>
      <td>SQL Fundamentals, SQL</td>
      <td>12</td>
      <td>Learn the most important PostgreSQL functions for manipulating, processing, and transforming data.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Intermediate SQL Server</td>
      <td>4</td>
      <td>2</td>
      <td>SQL Server Fundamentals, SQL</td>
      <td>13</td>
      <td>In this course, you will use T-SQL, the flavor of SQL used in Microsoft's SQL Server for data analysis.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Functions for Manipulating Data in SQL Server</td>
      <td>4</td>
      <td>2</td>
      <td>SQL Server Fundamentals, SQL</td>
      <td>13</td>
      <td>Learn the most important functions for manipulating, processing, and transforming data in SQL Server.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Time Series Analysis in SQL Server</td>
      <td>5</td>
      <td>2</td>
      <td>SQL Server Fundamentals, SQL</td>
      <td>13</td>
      <td>Explore ways to work with date and time data in SQL Server for time series analysis</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Introduction to Relational Databases in SQL</td>
      <td>4</td>
      <td>4</td>
      <td>SQL for Database Administrators, SQL</td>
      <td>16</td>
      <td>Learn how to create one of the most efficient ways of storing data - relational databases!</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Database Design</td>
      <td>4</td>
      <td>4</td>
      <td>SQL for Database Administrators, SQL</td>
      <td>16</td>
      <td>Learn to design databases in SQL.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Creating PostgreSQL Databases</td>
      <td>4</td>
      <td>1</td>
      <td>SQL for Database Administrators, SQL</td>
      <td>16</td>
      <td>This course teaches you the skills and knowledge necessary to create and manage your own PostgreSQL databases.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Improving Query Performance in PostgreSQL</td>
      <td>4</td>
      <td>1</td>
      <td>SQL for Database Administrators, SQL</td>
      <td>16</td>
      <td>Learn how to structure your PostgreSQL queries to run in a fraction of the time.</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Exploratory Data Analysis in SQL</td>
      <td>4</td>
      <td>2</td>
      <td>SQL for Business Analysts, SQL</td>
      <td>20</td>
      <td>Learn how to explore what's available in a database: the tables, relationships between them, and data stored in them.</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Data-Driven Decision Making in SQL</td>
      <td>4</td>
      <td>2</td>
      <td>SQL for Business Analysts, SQL</td>
      <td>20</td>
      <td>Learn how to analyze a SQL table and report insights to management.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    Spreadsheet:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Introduction to Statistics in Spreadsheets</td>
      <td>4</td>
      <td>2</td>
      <td>Intermediate Spreadsheets, Spreadsheet</td>
      <td>12</td>
      <td>Learn how to leverage statistical techniques using spreadsheets to more effectively work with and extract insights from your data.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Financial Analytics in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Finance Fundamentals, Spreadsheet</td>
      <td>12</td>
      <td>Learn how to build a graphical dashboard with spreadsheets to track the performance of financial securities.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Financial Modeling in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Finance Fundamentals, Spreadsheet</td>
      <td>12</td>
      <td>Learn basic business modeling including cash flows, investments, annuities, loan amortization, and more using Sheets.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Loan Amortization in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Finance Fundamentals, Spreadsheet</td>
      <td>12</td>
      <td>Learn how to build an amortization dashboard in spreadsheets with financial and conditional formulas.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Error and Uncertainty in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Intermediate Spreadsheets, Spreadsheet</td>
      <td>12</td>
      <td>Learn to distinguish real differences from random noise, and explore psychological crutches we use that interfere with our rational decision making.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Marketing Analytics in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Intermediate Spreadsheets, Spreadsheet</td>
      <td>12</td>
      <td>Learn how to ensure clean data entry and build dynamic dashboards to display your marketing data.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Data Analysis in Spreadsheets</td>
      <td>3</td>
      <td>1</td>
      <td>Spreadsheet Fundamentals, Spreadsheet</td>
      <td>15</td>
      <td>Learn how to analyze data with spreadsheets using functions such as SUM(), AVERAGE(), and VLOOKUP().</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Intermediate Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Spreadsheet Fundamentals, Spreadsheet</td>
      <td>15</td>
      <td>Expand your spreadsheets vocabulary by diving deeper into data types, including numeric data, logical data, and missing data.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Pivot Tables in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Spreadsheet Fundamentals, Spreadsheet</td>
      <td>15</td>
      <td>Explore the world of Pivot Tables within Google Sheets, and learn how to quickly organize thousands of data points with just a few clicks of the mouse.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Data Visualization in Spreadsheets</td>
      <td>4</td>
      <td>1</td>
      <td>Spreadsheet Fundamentals, Spreadsheet</td>
      <td>15</td>
      <td>Learn the fundamentals of data visualization using spreadsheets.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    Theory:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Data Visualization for Everyone</td>
      <td>2</td>
      <td>2</td>
      <td>Data Literacy Fundamentals, Theory</td>
      <td>10</td>
      <td>An introduction to data visualization with no coding involved.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Data Engineering for Everyone</td>
      <td>2</td>
      <td>2</td>
      <td>Data Literacy Fundamentals, Theory</td>
      <td>10</td>
      <td>Discover how data engineers lay the groundwork that makes data science possible. No coding involved!</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Data Science for Everyone</td>
      <td>2</td>
      <td>1</td>
      <td>Data Literacy Fundamentals, Theory</td>
      <td>10</td>
      <td>An introduction to data science with no coding involved.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Machine Learning for Everyone</td>
      <td>2</td>
      <td>1</td>
      <td>Data Literacy Fundamentals, Theory</td>
      <td>10</td>
      <td>An introduction to machine learning with no coding involved.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Cloud Computing for Everyone</td>
      <td>2</td>
      <td>1</td>
      <td>Data Literacy Fundamentals, Theory</td>
      <td>10</td>
      <td>A non-coding introduction to the world of cloud computing.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Introduction to Statistics in Spreadsheets</td>
      <td>4</td>
      <td>2</td>
      <td>Intermediate Spreadsheets, Spreadsheet</td>
      <td>12</td>
      <td>Learn how to leverage statistical techniques using spreadsheets to more effectively work with and extract insights from your data.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Data Science for Business</td>
      <td>2</td>
      <td>1</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Learn about data science and how can you use it to strengthen your organization.</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Machine Learning for Business</td>
      <td>2</td>
      <td>1</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Understand the fundamentals of Machine Learning and how it's applied in the business world.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Data-Driven Decision Making for Business</td>
      <td>2</td>
      <td>1</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Discover how to make better business decisions by applying practical data frameworks—no coding required.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Marketing Analytics for Business</td>
      <td>2</td>
      <td>1</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Discover how Marketing Analysts use data to understand customers and drive business growth.</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Introduction to Data Science in Python</td>
      <td>4</td>
      <td>3</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Dive into data science using Python and learn how to effectively analyze and visualize your data. No coding experience or skills needed.</td>
    </tr>
    <tr>
      <th>11</th>
      <td>AI Fundamentals</td>
      <td>4</td>
      <td>1</td>
      <td>Data Skills for Business, Theory</td>
      <td>20</td>
      <td>Learn the fundamentals of AI. No programming experience required!</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    Tableau:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Case Study: Analyzing Customer Churn in Tableau</td>
      <td>3</td>
      <td>1</td>
      <td>Tableau Fundamentals, Tableau</td>
      <td>25</td>
      <td>You will investigate a dataset from a fictitious company called Databel in Tableau, and need to figure out why customers are churning.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Creating Dashboards in Tableau</td>
      <td>4</td>
      <td>1</td>
      <td>Tableau Fundamentals, Tableau</td>
      <td>25</td>
      <td>Dashboards are a must-have in a data-driven world. Increase your impact on business performance with Tableau dashboards.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Connecting Data in Tableau</td>
      <td>4</td>
      <td>1</td>
      <td>Tableau Fundamentals, Tableau</td>
      <td>25</td>
      <td>Learn to connect Tableau to different data sources and prepare the data for a smooth analysis.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Introduction to Tableau</td>
      <td>6</td>
      <td>1</td>
      <td>Tableau Fundamentals, Tableau</td>
      <td>25</td>
      <td>Get started with Tableau, a widely used business intelligence (BI) and analytics software to explore, visualize, and securely share data.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Analyzing Data in Tableau</td>
      <td>8</td>
      <td>1</td>
      <td>Tableau Fundamentals, Tableau</td>
      <td>25</td>
      <td>Take your Tableau skills up a notch with advanced analytics and visualizations.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    R:
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Course</th>
      <th>Course Length</th>
      <th>Track Duplication</th>
      <th>Shortest Track</th>
      <th>Track Time Remaining</th>
      <th>Course Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Intermediate Data Visualization with ggplot2</td>
      <td>4</td>
      <td>2</td>
      <td>Data Visualization, R</td>
      <td>8</td>
      <td>Learn to use facets, coordinate systems and statistics in ggplot2 to create meaningful explanatory plots.</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Visualization Best Practices in R</td>
      <td>4</td>
      <td>1</td>
      <td>Data Visualization, R</td>
      <td>8</td>
      <td>Learn to  effectively convey your data with an overview of common charts, alternative visualization types, and perception-driven style enhancements.</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Introduction to Statistics in R</td>
      <td>4</td>
      <td>4</td>
      <td>Data Analyst, R</td>
      <td>12</td>
      <td>Grow your statistical skills and learn how to collect, analyze, and draw accurate conclusions from data.</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Joining Data with dplyr</td>
      <td>4</td>
      <td>3</td>
      <td>Data Analyst, R</td>
      <td>12</td>
      <td>Learn to combine data across multiple tables to answer more complex questions with dplyr.</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Data Manipulation with R</td>
      <td>4</td>
      <td>3</td>
      <td>Data Analyst, R</td>
      <td>12</td>
      <td>Learn how to efficiently collect and download data from any website using R.</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Writing Efficient R Code</td>
      <td>4</td>
      <td>3</td>
      <td>R Programming, R</td>
      <td>12</td>
      <td>Learn to write faster R code, discover benchmarking and profiling, and unlock the secrets of parallel programming.</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Introduction to Writing Functions in R</td>
      <td>4</td>
      <td>3</td>
      <td>R Programming, R</td>
      <td>12</td>
      <td>Use cluster analysis to glean insights into cryptocurrency gambling behavior.</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Unsupervised Learning in R</td>
      <td>4</td>
      <td>3</td>
      <td>Unsupervised Machine Learning, R</td>
      <td>12</td>
      <td>This course provides an intro to clustering and dimensionality reduction in R from a machine learning perspective.</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Cluster Analysis in R</td>
      <td>4</td>
      <td>2</td>
      <td>Unsupervised Machine Learning, R</td>
      <td>12</td>
      <td>Develop a strong intuition for how hierarchical and k-means clustering work and learn how to apply them to extract insights from your data.</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Factor Analysis in R</td>
      <td>4</td>
      <td>2</td>
      <td>Unsupervised Machine Learning, R</td>
      <td>12</td>
      <td>Explore latent variables, such as personality using exploratory and confirmatory factor analyses.</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Object-Oriented Programming with S3 and R6 in R</td>
      <td>4</td>
      <td>1</td>
      <td>R Programming, R</td>
      <td>12</td>
      <td>Manage the complexity in your code using object-oriented programming with the S3 and R6 systems.</td>
    </tr>
  </tbody>
</table>
</div>


    
    
    

## Conclusion

The DC Recommender shows a good amount of useful information about suitable courses and tracks. Regarding practical use, it may just be used to point me in the direction of appropriate tracks, but even that is useful and can help as a motivator. This notebook has been written such that certain chunks of code can be commented out to avoid unecessary scraping/processing, and commented back in to allow refreshing the cached data.

All-in-all, I believe this is a useful script to direct my learning focus, through use of web scraping, data engineering, and production of a bespoke recommendation engine. However, if it seems for a moment that this may have been just a big project to organise doing some courses rather than actually doing them, well I'll let you know my progress!
