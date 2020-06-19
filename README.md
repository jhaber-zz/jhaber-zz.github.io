# Web-crawling the (figurative) Wild West, Step 1: Gathering URLs
by [Jaren Haber, PhD](https://www.jarenhaber.com) <br>
email: jhaber@berkeley.edu


## Introduction
Digital era, especially given shelter in place. Say you have a population of something you want to track: organizations like schools or hospitals, individuals like politicians or meme generators (name? culture producers?), any kind of community that leaves a digital footprint online. How do you find out where they are? 

As is often the case in our Internet-savvy era, the answer is simply: Check Google. But hold your horses, because manual Google searching is slow to do at scale, in no small part because in not every case do the most valuable matches appear at the top of results page. Google enumerates all kinds of information from all kinds of places, including not only the original websites of your online entities of interest, but also of third-party content collectors, from  like Trulia, Yelp, and YellowPages (others?).
In addition, if your list of entities is imperfect--perhaps deriving from an even slightly out of date directory--then some of your searches will find nothing valuable. The result is messy information gathering made noisy by false positives (when your search algorithm finds some match but shouldn't, e.g. for entities that no longer exist) and false negatives (when your search algorithm should find a quality match but doesn't, e.g. when it returns a third-party site instead of an original one). This can make locating your online population a truly hairy endeavor, and before long you may ask yourself: Why not just drink a ton of Red Bulls and do this by hand?

Not so fast! Not only would that wreak havoc on your sleep rhythms (a more common problem than you might think [link]), but also you're throwing in the towel too fast. There are ways to get around these obstacles and craft a fast, automated search algorithm optimized for your specific population that will tell you their online information. This blog will tell you how, emphasizing URLs in particular--the central piece of information to enable web-crawling at scale. I will use the corpus of 6,853 (check this) charter schools as a case study. A perfect one, really, given that they are super heterogeneous and some 300 (check this) close every year. If I can gather a comprehensive, up-to-date list of URLs for this messy, decentralized group, then you could apply this method to almost anything with an online presence (especially a website).

At the end of the day, some hand-cleaning of your resulting digital information is still necessary. However, implementing a robust search method with several robustness checks minimizes the need for this painstaking effort.

Next is web-crawling. Maybe a future blog-post. Or give a brief guide here! (what I sent to bay-sicss folks) 


## Technical notes?
This script uses two related functions to scrape the best URL from online sources: 
> The Google Places API. See the [GitHub page](https://github.com/slimkrazy/python-google-places) for the Python wrapper and sample code, [Google Web Services](https://developers.google.com/places/web-service/) for general documentation, and [here](https://developers.google.com/places/web-service/details) for details on Place Details requests.

> The Google Search function (manually filtered). See [here](https://pypi.python.org/pypi/google) for source code and [here](http://pythonhosted.org/google/) for documentation.

To get an API key for the Google Places API (or Knowledge Graph API), go to the [Google API Console](http://code.google.com/apis/console).
To upgrade your quota limits, sign up for billing--it's free and raises your daily request quota from 1K to 150K (!!).

The code below doesn't use Google's Knowledge Graph (KG) Search API because this turns out NOT to reveal websites related to search results (despite these being displayed in the KG cards visible at right in a standard Google search). The KG API is only useful for scraping KG id, description, name, and other basic/ irrelevant info. TO see examples of how the KG API constructs a search URL, etc., (see [here](http://searchengineland.com/cool-tricks-hack-googles-knowledge-graph-results-featuring-donald-trump-268231)).

Possibly useful note on debugging: An issue causing the GooglePlaces package to unnecessarily give a "ValueError" and stop was resolved in [July 2017](https://github.com/slimkrazy/python-google-places/issues/59). <br>
Other instances of this error may occur if Google Places API cannot identify a location as given. Dealing with this is a matter of proper Exception handling (which seems to be working fine below).


## Initialize Python search environment

Easy to install packages in Jupyter notebook:
```
# Install packages
!pip install google # For automated Google searching 
!pip install https://github.com/slimkrazy/python-google-places/zipball/master # Google Places API
```
```
# Import packages
import googlesearch  # automated Google Search package
from googleplaces import GooglePlaces, types, lang  # Google Places API

import csv, re, os  # Standard packages
import pandas as pd  # for working with csv files
import urllib, requests  # for scraping
```
```
# Initialize Google Places API search functionality
places_api_key = re.sub("\n", "", open("../data/places_api_key.txt").read())
print(places_api_key)

google_places = GooglePlaces(places_api_key)
```
```
# Define helper functions
def dicts_to_csv(list_of_dicts, file_name, header):
    '''This helper function writes a list of dictionaries to a csv called file_name, with column names decided by 'header'.'''
    
    with open(file_name, 'w') as output_file:
        print("Saving to " + str(file_name) + " ...")
        dict_writer = csv.DictWriter(output_file, header)
        dict_writer.writeheader()
        dict_writer.writerows(list_of_dicts)

def count_left(list_of_dicts, varname):
    '''This helper function determines how many dicts in list_of_dicts don't have a valid key/value pair with key varname.'''
    
    count = 0
    for school in list_of_dicts:
        if school[varname] == "" or school[varname] == None:
            count += 1

    print(str(count) + " schools in this data are missing " + str(varname) + "s.")
```

Here's a list of sites we DON'T want to spider, but that an automated Google search might return... and we might thus accidentally spider unless we filter them out (as below)!

```
# Define black-list of websites
bad_sites = []
with open('../data/bad_sites.csv', 'r', encoding = 'utf-8') as csvfile:
    for row in csvfile:
        bad_sites.append(re.sub('\n', '', row))

print(bad_sites)
```

For my corpus, here's the output:
```
['high-schools.com', 'yelp.com', 'har.com', 'trulia.com', 'redfin.com', 'practutor.com', 'startclass.com', 'greatschools.org', 'greatschools.com', 'greatschools.net', 'paschoolperformance.org', 'worldcontactinfo.com', 'kula.com', 'mapquest.com', 'maps.net', 'google.com', 'facebook.com', 'zillow.com', 'manta.com', 'yellowpages.com', 'usnews.com', 'publicschoolreview.com', 'publicschoolreview.org', 'schooldigger.com', 'niche.com', 'privateschoolreview.com', 'cappex.com', 'collegeconfidential.com', 'tripsadvisor.com', 'groupon.com', 'school-ratings.com', 'superpages.com', 'onsaleph.com', 'psk12.com', 'schoolmatters.com', 'neighborhoodscout.com', 'localschooldirectory.com', 'publicschoolsk12.com', 'schooldatadirect.org', 'nces.ed.gov', 'cityrating.com', 'blogspot.com', 'public-schools.findthebest.com', 'twitter.com', 'zoominfo.com', 'jigsaw.com', 'hoovers.com', 'corporateinformation.com', 'doe.k12.ga.us', 'gradeschools.net', 'charterschoolratings.net', 'schools.net', 'insiderpages.com', 'parentstown.com', 'freepreschools.org', 'fresno.schools.net', 'baldwin.school.org', 'illinoisschools.com', 'seattleprogressiveschools.org', 'schoolchoiceintl.com', 'ratemyteachers.com', 'ade.az.gov', 'cde.ca.gov']
```

## Get sample output

See the Google Places API wrapper at work!
```
# Get sample results
school_name = "River City Scholars Charter Academy"
address = "944 Evergreen Street, Grand Rapids, MI 49507"

query_result = google_places.nearby_search(
        location=address, name=school_name,
        radius=15000, types=[types.TYPE_SCHOOL], rankby='distance')

for place in query_result.places:
    print(place.name)
    place.get_details()  # makes further API call
    #print(place.details) # A dict matching the JSON response from Google.
    print(place.website)
    print(place.formatted_address)

# Are there any additional pages of results?
if query_result.has_next_page_token:
    query_result_next_page = google_places.nearby_search(
            pagetoken=query_result.next_page_token)
```
Output:
```
```

Text
```
# Example of using the google search function:
for url in search('DR DAVID C WALKER INT 6500 IH 35 N STE C, SAN ANTONIO, TX 78218', \
                  stop=5, pause=5.0):
    print(url)
```
Output:
```
https://www.har.com/school/015806106/dr-david-c-walker-elementary-school
https://www.excellence-sa.org/walker
https://www.schooldigger.com/go/TX/schools/0006211404/school.aspx
https://www.greatschools.org/texas/san-antonio/12035-Dr-David-C-Walker-Intermediate-School/
https://www.greatschools.org/texas/san-antonio/12035-Dr-David-C-Walker-Intermediate-School/#Test_scores
https://www.greatschools.org/texas/san-antonio/12035-Dr-David-C-Walker-Intermediate-School/#Low-income_students
https://www.greatschools.org/texas/san-antonio/12035-Dr-David-C-Walker-Intermediate-School/#Students_with_Disabilities
https://www.niche.com/k12/dr-david-c-walker-intermediate-school-san-antonio-tx/
https://www.facebook.com/pages/Dr-David-C-Walker-Int/598905323548274
https://www.publicschoolreview.com/dr-david-c-walker-elementary-school-profile
```


## Read in data

sample = []  # make empty list in which to store the dictionaries

if os.path.exists('../data/sample.csv'):  # first, check if file containing search results is available on disk
    file_path = '../data/sample.csv'
else:  # use original data if no existing results are available on disk
    file_path = '../../data_management/data/charters_unscraped_noURL_2015.csv'

with open(file_path, 'r', encoding = 'utf-8') as csvfile: # open file                      
    print('Reading in ' + str(file_path) + ' ...')
    reader = csv.DictReader(csvfile)  # create a reader
    for row in reader:  # loop through rows
        sample.append(row)  # append each row to the list

print("\nColumns in data: ")
list(sample[0])

# Create new "URL" and "NUM_BAD_URLS" variables for each school, without overwriting any with data there already:
for school in sample:
    try:
        if len(school["URL"]) > 0:
            pass
        
    except (KeyError, NameError):
        school["URL"] = ""

for school in sample:
    try:
        if school["NUM_BAD_URLS"]:
            pass
        
    except (KeyError, NameError):
        school["NUM_BAD_URLS"] = ""
            
#### Take a look at the first entry's contents and the variables list in our sample (a list of dictionaries)
print(sample[1]["SEARCH16"], "\n", sample[1]["URL"], "\n", sample[1]["ADDRESS16"], "\n")
print(sample[1].keys())


## Get URLs

def getURL(school_name, address, bad_sites_list): # manual_url
    
    '''This function finds the one best URL for a school using two methods:
    
    1. If a school with this name can be found within 20 km (to account for proximal relocations) in
    the Google Maps database (using the Google Places API), AND
    if this school has a website on record, then this website is returned.
    If no school is found, the school discovered has missing data in Google's database (latitude/longitude, 
    address, etc.), or the address on record is unreadable, this passes to method #2. 
    
    2. An automated Google search using the school's name + address. This is an essential backup plan to 
    Google Places API, because sometimes the address on record (courtesy of Dept. of Ed. and our tax dollars) is not 
    in Google's database. For example, look at: "3520 Central Pkwy Ste 143 Mezz, Cincinnati, OH 45223". 
    No wonder Google Maps can't find this. How could it intelligibly interpret "Mezz"?
    
    Whether using the first or second method, this function excludes URLs with any of the 62 bad_sites defined above, 
    e.g. trulia.com, greatschools.org, mapquest. It returns the number of excluded URLs (from either method) 
    and the first non-bad URL discovered.'''
    
    
    ## INITIALIZE
    
    new_urls = []    # start with empty list
    good_url = ""    # output goes here
    k = 0    # initialize counter for number of URLs skipped
    
    radsearch = 15000  # define radius of Google Places API search, in km
    numgoo = 20  # define number of google results to collect for method #2
    wait_time = 20.0  # define length of pause between Google searches (longer is better for big catches like this)
    
    search_terms = school_name + " " + address
    print("Getting URL for " + school_name + ", " + address + "...")    # show school name & address
    
    
    
    ## FIRST URL-SCRAPE ATTEMPT: GOOGLE PLACES API
    # Search for nearest school with this name within radsearch km of this address
    
    try:
        query_result = google_places.nearby_search(
            location=address, name=school_name,
            radius=radsearch, types=[types.TYPE_SCHOOL], rankby='distance')
        
        for place in query_result.places:
            place.get_details()  # Make further API call to get detailed info on this place

            found_name = place.name  # Compare this name in Places API to school's name on file
            found_address = place.formatted_address  # Compare this address in Places API to address on file

            try: 
                url = place.website  # Grab school URL from Google Places API, if it's there

                if any(domain in url for domain in bad_sites_list):
                    k+=1    # If this url is in bad_sites_list, add 1 to counter and move on
                    #print("  URL in Google Places API is a bad site. Moving on.")

                else:
                    good_url = url
                    print("    Success! URL obtained from Google Places API with " + str(k) + " bad URLs avoided.")
                    
                    '''
                    # For testing/ debugging purposes:
                    
                    print("  VALIDITY CHECK: Is the discovered URL of " + good_url + \
                          " consistent with the known URL of " + manual_url + " ?")
                    print("  Also, is the discovered name + address of " + found_name + " " + found_address + \
                          " consistent with the known name/address of: " + search_terms + " ?")
                    
                    if manual_url != "":
                        if manual_url == good_url:
                            print("    Awesome! The known and discovered URLs are the SAME!")
                    '''
                            
                    return(k, good_url)  # Returns valid URL of the Place discovered in Google Places API
        
            except:  # No URL in the Google database? Then try next API result or move on to Google searching.
                print("  Error collecting URL from Google Places API. Moving on.")
                pass
    
    except:
        print("  Google Places API search failed. Moving on to Google search.")
        pass
    
    

    ## SECOND URL-SCRAPE ATTEMPT: FILTERED GOOGLE SEARCH
    # Automate Google search and take first result that doesn't have a bad_sites_list element in it.
    
    
    # Loop through google search output to find first good result:
    try:
        new_urls = list(search(search_terms, stop=numgoo, pause=wait_time))  # Grab first numgoo Google results (URLs)
        print("  Successfully collected Google search results.")
        
        for url in new_urls:
            if any(domain in url for domain in bad_sites_list):
                k+=1    # If this url is in bad_sites_list, add 1 to counter and move on
                #print("  Bad site detected. Moving on.")
            else:
                good_url = url
                print("    Success! URL obtained by Google search with " + str(k) + " bad URLs avoided.")
                break    # Exit for loop after first good url is found
                
    
    except:
        print("  Problem with collecting Google search results. Try this by hand instead.")
            
        
    '''
    # For testing/ debugging purposes:
    
    if k>2:  # Print warning messages depending on number of bad sites preceding good_url
        print("  WARNING!! CHECK THIS URL!: " + good_url + \
              "\n" + str(k) + " bad Google results have been omitted.")
    if k>1:
        print(str(k) + " bad Google results have been omitted. Check this URL!")
    elif k>0:
        print(str(k) + " bad Google result has been omitted. Check this URL!")
    else: 
        print("  No bad sites detected. Reliable URL!")
    
    if manual_url != "":
        if manual_url == good_url:
            print("    Awesome! The known and discovered URLs are the SAME!")
    '''
    
    if good_url == "":
        print("  WARNING! No good URL found via API or google search.\n")
    
    return(k, good_url)
    

numschools = 0  # initialize scraping counter

keys = sample[0].keys()  # define keys for writing function
fname = "../data/sample.csv"  # define file name for writing function

for school in sample[:2]:
    print(school["URL"])
    

Now to call the above function and actually scrape these things!
```
for school in sample: # loop through list of schools
    if school["URL"] == "":  # if URL is missing, fill that in by scraping
        numschools += 1
        school["NUM_BAD_URLS"], school["URL"] = getURL(school["SCHNAM16"], school["ADDRESS16"], bad_sites) # school["MANUAL_URL"]
    
    else:
        if school["URL"]:
            pass  # If URL exists, don't bother scraping it again

        else:  # If URL hasn't been defined, then scrape it!
            numschools += 1
            school["NUM_BAD_URLS"], school["URL"] = "", "" # start with empty strings
            school["NUM_BAD_URLS"], school["URL"] = getURL(school["SCHNAM16"], school["ADDRESS16"], bad_sites) # school["MANUAL_URL"]

print("\n\nURLs discovered for " + str(numschools) + " schools.")
```

In summer 2017, the above approach works to get a good URL for 6,677 out of the 6,752 schools in this data set. Not bad! <br>
For some reason, the Google search algorithm (method #2) is less likely to work after passing from the Google Places API. <br>
To fill in for the remaining 75, let's skip the function's layers of code and just call the google search function by hand.

```
for school in sample:
    if school["URL"] == "":
        k = 0  # initialize counter for number of URLs skipped
        school["NUM_BAD_URLS"] = ""

        print("Scraping URL for " + school["SEARCH"] + "...")
        urls_list = list(search(school["SEARCH"], stop=20, pause=10.0))
        print("  URLs list collected successfully!")

        for url in urls_list:
            if any(domain in url for domain in bad_sites):
                k+=1    # If this url is in bad_sites_list, add 1 to counter and move on
                # print("  Bad site detected. Moving on.")
            else:
                good_url = url
                print("    Success! URL obtained by Google search with " + str(k) + " bad URLs avoided.")

                school["URL"] = good_url
                school["NUM_BAD_URLS"] = k
                
                count_left(sample, 'URL')
                dicts_to_csv(sample, fname, keys)
                print()
                break    # Exit for loop after first good url is found                               
                                           
    else:
        pass
```
```
count_left(sample, 'URL')
dicts_to_csv(sample, fname, keys)
```
# CHECK OUT RESULTS
# TO DO: Make a histogram of 'NUM_BAD_URLS'
# systematic way to look at problem URLs (with k > 0)?

f = 0
for school in sample:
    if int(school['NUM_BAD_URLS']) > 14:
        print(school["SEARCH"], "\n", school["URL"], "\n")
        f += 1

print(str(f))


## Further reading
Notebook on examining long scripts. Maybe look at how I did this in spreadsheet!
