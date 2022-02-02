# google-scholar-scrapy-spider

Python Scrapy spider that searches Google Scholar for a particular keyword and extracts all search results from the product page. The spider will iterate through all pages returned by the keyword query. The following are the fields the spider scrapes for the Google Scholar search results page:

- Title 
- Link
- Citations
- Related Links
- Number of Verions
- Author
- Publisher
- Snippet

This Google Scholar spider uses [Scraper API](https://www.scraperapi.com/) as the proxy solution. Scraper API has a free plan that allows you to make up to 1,000 requests per month which makes it ideal for the development phase, but can be easily scaled up to millions of pages per month if needs be.

To monitor the scraper, this scraper uses [ScrapeOps](https://scrapeops.io/). **Live demo here:** [ScrapeOps Demo](https://scrapeops.io/app/login/demo)

![ScrapeOps Dashboard](https://scrapeops.io/assets/images/scrapeops-promo-286a59166d9f41db1c195f619aa36a06.png)


## Using the Google Scholar Spider
Make sure Scrapy is installed:

```
pip install scrapy
```

Set the keywords you want to search in Google Scholar.

```
queries = ['airbnb', 'covid-19']
```

### Setting Up ScraperAPI
Signup to [Scraper API](https://www.scraperapi.com/signup) and get your free API key that allows you to scrape 1,000 pages per month for free. Enter your API key into the API variable:

```
API_KEY = '<YOUR_API_KEY>'

def get_url(url):
    payload = {'api_key': API_KEY, 'url': url, 'country_code': 'us'}
    proxy_url = 'http://api.scraperapi.com/?' + urlencode(payload)
    return proxy_url
```

By default, the spider is set to have a max concurrency of 5 concurrent requests as this the max concurrency allowed on Scraper APIs free plan. If you have a plan with higher concurrency then make sure to increase the max concurrency in the `settings.py`.

```
## settings.py

CONCURRENT_REQUESTS = 5
RETRY_TIMES = 5

# DOWNLOAD_DELAY
# RANDOMIZE_DOWNLOAD_DELAY
```

We should also set `RETRY_TIMES` to tell Scrapy to retry any failed requests (to 5 for example) and make sure that `DOWNLOAD_DELAY`  and `RANDOMIZE_DOWNLOAD_DELAY` aren’t enabled as these will lower your concurrency and are not needed with Scraper API.

### Integrating ScrapeOps
[ScrapeOps](https://scrapeops.io/) is already integrated into the scraper via the `settings.py` file. However, to use it you must:

Install the [ScrapeOps Scrapy SDK](https://github.com/ScrapeOps/scrapeops-scrapy-sdk) on your machine.

```
pip install scrapeops-scrapy
```

And sign up for a [free ScrapeOps account here](https://scrapeops.io/app/register) so you can insert your **API Key** into the `settings.py` file:

```
    ## settings.py
    
    ## Add Your ScrapeOps API key
    SCRAPEOPS_API_KEY = 'YOUR_API_KEY'
    
    ## Add In The ScrapeOps Extension
    EXTENSIONS = {
     'scrapeops_scrapy.extension.ScrapeOpsMonitor': 500, 
    }
    
    ## Update The Download Middlewares
    DOWNLOADER_MIDDLEWARES = { 
	'scrapeops_scrapy.middleware.retry.RetryMiddleware': 550, 
	'scrapy.downloadermiddlewares.retry.RetryMiddleware': None, 
    }
```
From there, our scraping stats will be automatically logged and automatically shipped to our dashboard.

### Running The Spider
To run the spider, use:

```
scrapy crawl scholar -o test.csv
```

## Editing the Google Scholar Spider
The spider has 3 parts:

1. **start_requests -** will construct the Google Scholar URL for the search queries and send the request to Google.
2. **parse -** will extract all the search results from the Google Scholar search results.
3. **get_url -** to scrape Google Scholar at scale without getting blocked we need to use a proxy solution. For this project we will use Scraper API so we need to create a function to send the request to their API endpoint.

 If you want to scrape more or less fields from the search results page then edit the XPath selectors in the **parse** function:
 
 ```
 def parse(self, response):
        print(response.url)
        position = response.meta['position']
        for res in response.xpath('//*[@data-rp]'):
            link = res.xpath('.//h3/a/@href').extract_first()
            temp = res.xpath('.//h3/a//text()').extract()
            if not temp:
                title = "[C] " + "".join(res.xpath('.//h3/span[@id]//text()').extract())
            else:
                title = "".join(temp)
            snippet = "".join(res.xpath('.//*[@class="gs_rs"]//text()').extract())
            cited = res.xpath('.//a[starts-with(text(),"Cited")]/text()').extract_first()
            temp = res.xpath('.//a[starts-with(text(),"Related")]/@href').extract_first()
            related = "https://scholar.google.com" + temp if temp else ""
            num_versions = res.xpath('.//a[contains(text(),"version")]/text()').extract_first()
            published_data = "".join(res.xpath('.//div[@class="gs_a"]//text()').extract())
            position += 1
            item = {'title': title, 'link': link, 'cited': cited, 'relatedLink': related, 'position': position,
                    'numOfVersions': num_versions, 'publishedData': published_data, 'snippet': snippet}
            yield item
        next_page = response.xpath('//td[@align="left"]/a/@href').extract_first()
        if next_page:
            url = "https://scholar.google.com" + next_page
            yield scrapy.Request(get_url(url), callback=self.parse,meta={'position': position})
```

If you don't want to scrape every page returned for that keyword then comment out the `next_page` section of the **parse** function.
