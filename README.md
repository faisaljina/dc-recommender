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




    ([], [], [])



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
# Show the first few skill tracks
skill_tracks[:4]
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
      'https://www.datacamp.com/tracks/data-manipulation-with-r')]



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

# Finally, combine the names, technologies and urls into a list of tuples
career_tracks = list(zip(track_names, track_techs, track_urls))
# Show the first few career tracks
career_tracks[:4]
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
      'https://www.datacamp.com/tracks/data-scientist-with-python')]




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
# Check the first few lines
all_tracks[:4]
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
      'https://www.datacamp.com/tracks/data-manipulation-with-r\n']]



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

# Show all courses by track and technology (output hidden here due to length)
for tech in technologies:
    show_courses(tech)
```

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
# Show the first few tracks and their technologies
{k: tracks_tech[k] for k in list(tracks_tech)[:4]}
```




    {'R Programming, R': 'R',
     'Importing & Cleaning Data, R': 'R',
     'Data Visualization, R': 'R',
     'Data Manipulation, R': 'R'}




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
# Show the first few courses and their counts
{k: course_counts[k] for k in list(course_counts)[:4]}
```




    {'Intermediate Regression in R': 5,
     'Data Manipulation with dplyr ': 4,
     'Introduction to Statistics in R': 4,
     'Data Manipulation with pandas': 4}




```python
# Create a dictionary of courses and the tracks they are in
course_tracks = dict()
for key in complete_tracks:
    for course in complete_tracks[key]:
        if course in course_tracks:
            course_tracks[course].append(key)
        else:
            course_tracks[course] = [key]
# Show the first few courses and their tracks
{k: course_tracks[k] for k in list(course_tracks)[:4]}
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
      'R Programmer, R']}




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
# Show the first few tracks and their total duration
{k: track_durations[k] for k in list(track_durations)[:4]}
```




    {'Data Literacy Fundamentals, Theory': 10,
     'Data Visualization, R': 12,
     'Unsupervised Machine Learning, R': 12,
     'Image Processing, Python': 12}



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

# Show the first few completed courses and the total number
print("Completed courses:", dc_done_list[:4])
print("Total number of completed courses:", len(dc_done_list))
```

    
    Completed courses: ['ARIMAModelsinR', 'CaseStudiesManipulatingTimeSeriesDatainR', 'Data Manipulation with pandas', 'DataManipulationwithdplyr']
    
    Total number of completed courses: 24
    


```python
# Get list of all possible DataCamp courses
all_courses = []
for key in complete_tracks:
    for course in complete_tracks[key]:
        all_courses.append(course)
all_courses = set(all_courses)
# Show the first few courses and the total number
print({k for k in list(all_courses)[:4]})
print("Total number of courses:", len(all_courses))
```

    {'Introduction to R for Finance', 'Introduction to Statistics in R', 'Financial Analytics in Spreadsheets', 'Preprocessing for Machine Learning in Python'}
    Total number of courses: 289
    


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

# Show the first few completed courses as matched with the scraped list
completed_courses[:4]
```

    List is complete
    Completed: 24
    




    ['ARIMA Models in R',
     'Case Studies: Manipulating Time Series Data in R',
     'Data Manipulation with dplyr ',
     'Data Manipulation with pandas']




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
# Show the first few tracks and their total duration
{k: track_remaining_durations[k] for k in list(track_remaining_durations)[:4]}
```




    {'Data Visualization, R': 8,
     'Data Literacy Fundamentals, Theory': 10,
     'R Programming, R': 12,
     'Data Manipulation, Python': 12}



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

# Show the first few courses for Python and the total number of courses found
print(get_courses_by_tech("Python", k_tracks=5)[:4])
print("Total number of courses:", len(get_courses_by_tech("Python", k_tracks=5)))
```

    [('Joining Data with pandas', 4), ('Analyzing Police Activity with pandas', 4), ('Introduction to Databases in Python', 4), ('Image Processing in Python', 4)]
    Total number of courses: 17
    

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

# Check a course to find the track and remaining duration
get_track_by_course('Introduction to Statistics in Spreadsheets')
```




    ('Intermediate Spreadsheets, Spreadsheet', 12)



Last couple of things now, just want to double-check I can find the number of tracks a course appears in.


```python
# Check no. of tracks a course is in
test_course = 'Joining Data with pandas'
print("Number of tracks:", course_counts[test_course])
```

    Number of tracks: 3
    

Great, and just as some added sugar, I'd like to have a dictionary of courses and their descriptions.


```python
# Create a dictionary of all courses and their descriptions
course_descriptions = dict()
for track in complete_tracks:
    for course, description in complete_tracks[track].items():
        course_descriptions[course] = description[0]

# Show the first few courses and their descriptions
{k: course_descriptions[k] for k in list(course_descriptions.keys())[:4]}
```




    {'Introduction to R': 'Master the basics of data analysis by manipulating common data structures such as vectors, matrices, and data frames.',
     'Intermediate R': 'Continue your journey to becoming an R ninja by learning about conditional statements, loops, and vector functions.',
     'Writing Efficient R Code': 'Learn to write faster R code, discover benchmarking and profiling, and unlock the secrets of parallel programming.',
     'Introduction to Writing Functions in R': 'Use cluster analysis to glean insights into cryptocurrency gambling behavior.'}



### Put it all together

A dataframe is a suitable output for my recommendations as they display nicely in VSCode. Let's put all the above functions together and see what we get.


```python
# Function taking a technology and performing the above functions to get the courses, tracks and remaining durations
def get_recommendations(tech = None, k_tracks = 5, row_limit = 10)-> pd.DataFrame:
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
    

get_recommendations(k_tracks=3, row_limit=10)

```

    Choose a technology from:
    ['Theory', 'Spreadsheet', 'Power BI', 'SQL', 'R', 'Tableau', 'Python']
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
      <td>Learn to combine data from multiple tables by ...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Analyzing Police Activity with pandas</td>
      <td>4</td>
      <td>2</td>
      <td>Data Manipulation, Python</td>
      <td>12</td>
      <td>Learn to draw conclusions from limited data us...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Image Processing in Python</td>
      <td>4</td>
      <td>2</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn powerful techniques for image analysis i...</td>
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
      <td>In this course, you'll learn the basics of rel...</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Recurrent Neural Networks for Language Modelin...</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Use RNNs to classify text sentiment, generate ...</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Machine Translation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Are you curious about the inner workings of th...</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Natural Language Generation in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Deep Learning for NLP, Python</td>
      <td>12</td>
      <td>Imitate Shakespear, translate language and aut...</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Biomedical Image Analysis in Python</td>
      <td>4</td>
      <td>1</td>
      <td>Image Processing, Python</td>
      <td>12</td>
      <td>Learn the fundamentals of exploring, manipulat...</td>
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
