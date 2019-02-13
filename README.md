# Web Scraping and Document Databases

This web application scrapes various websites for data related to the Mission to Mars and displays the information in a single HTML page.

## Scraping

Starting in Jupyter notebook this web application uses Splinter, to navigate the websites when needed, and BeautifulSoup, to help find and parse out the necessary data. 

The executable path is set and initializes the chrome browser in splinter. The chromedriver path can be found by `!which chromedriver`; the executable path needs to be added to the local hosts' `PATH` under environmet variables, which can be found in the System's advanced settings.

```
executable_path = {'executable_path': '/usr/local/bin/chromedriver'}
browser = Browser('chrome', **executable_path)
```

The Mars web application scrapes data from five different websites to scrape: 

```
1. A news article and title 
2. Separate images of the different hemispheres in URL format 
3. A large image of mars in URL format
4. Most recent tweet from the mars twitter account `Mars Weather` 
5. A dataframe containing a description
```

Each URL is set to a variable and uses `Browser from Splinter` to navigate each website. 

```
url = 'https://mars.nasa.gov/news/'
browser.visit(url)
```

### Nasa mars news site

After defining the URL variable, the browser html is converted into a soup object. This allows the developer to extract the information previously mentioned. Since the `find()` function outputs the entire string from the source's HTML, the `get_text()` is added and stored into the `news_title` variable. These steps are repeated in order to extract the content of the article as well.

```
slide_elem = news_soup.select_one('ul.item_list li.slide')
slide_elem.find("div", class_='content_title')

news_title = slide_elem.find("div", class_='content_title').get_text()
```

### Mars Weather

An empty dictionary is created,`hemisphere_image_urls = []`, in order to store the URLS for the different hemispheres. Once the list of all of the hemispheres are extracted:

```
links = browser.find_by_css("a.product-item h3")
```

Next,  a for loop is created to go through the links, click each link, find the sample anchor, and return the href. In order to avoid a stale element exception, the elements must be found at the beginning of each loop. At the end of each loop the hemisphere URL is  appended to the dictionary.

```
for i in range(len(links)):
    hemisphere = {}
    browser.find_by_css("a.product-item h3")[i].click()
    sample_elem = browser.find_link_by_text('Sample').first
    hemisphere['img_url'] = sample_elem['href']
    hemisphere['title'] = browser.find_by_css("h2.title").text
    hemisphere_image_urls.append(hemisphere)
    browser.back()
```

### Mars Facts

Pandas is imported in order to create a dataframe that is able to store integers reflecting a summary of facts about Mars. 

```
import pandas as pd
df = pd.read_html('http://space-facts.com/mars/')[0]
df.columns=['description', 'value']
df.set_index('description', inplace=True)
```

## MongoDB and Flash Application

Uses MongoDB with Flask templating to create a new HTML page that displays all of the information that was scraped from the URLs above. Starts by converting the Jupyter notebook into a Python script called `scrape_mars.py` with a function called `scrape` that will execute all of the scraping code from above and return one Python dictionary containing all of the scraped data. 

```
data = {
        "news_title": news_title,
        "news_paragraph": news_paragraph,
        "featured_image": featured_image(browser),
        "hemispheres": hemispheres(browser),
        "weather": twitter_weather(browser),
        "facts": mars_facts(),
        "last_modified": dt.datetime.now()
    }
```

In `app.py` a root route `/` is created that will query the Mongo database and pass the mars data into an HTML template to display the data.
 
 ```
 @app.route("/")
def index():
    mars = mongo.db.mars.find_one()
    return render_template("index.html", mars=mars)
```

A route called `/scrape` is created that imports the `scrape_mars.py` script and calls the `scrape` function. Stores the return value in Mongo as a Python dictionary.

```
@app.route("/scrape")
def scrape():
    mars = mongo.db.mars
    mars_data = scrape_mars.scrape_all()
    mars.update({}, mars_data, upsert=True)
    return "Scraping Successful!"
```

