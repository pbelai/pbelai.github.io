# Scraping data from website with infinite scrolling


R is a popular language for scraping websites and has plenty of packages for scraping static websites. However, dynamically generated websites are growing in popularity. These are harder to scrape, as the content is generated after we load the website or do some events on the website. Luckily, R has a solution even for this. Package `RSelenium` which enables us to connect to the Selenium server. You can learn more about this package from [its vignette](https://cran.r-project.org/web/packages/RSelenium/vignettes/basics.html). In this tutorial, I am going to show you, how to scrape websites with infinite scrolling.

# RSelenium preparation

And what better place to scrape, than the one we like to waste time on. That's right we are going to scrape data from youtube. First of all, it is polite to check if the scraping is allowed. We can check `robots.txt` for youtube to see if youtube is fine with us, scraping it. As there are no restrictions we are fine to continue.

As a first step, we need to start the Selenium server and browser and open it. We will use headless Firefox, as we do not need to see, what is happening and we want to scrape in the background. We do all this with the following code:


```r
rD <- RSelenium::rsDriver(port = 4485L,
    browser = "firefox",
    extraCapabilities = list(
        "moz:firefoxOptions" = list(
            args = list('--headless')
        )
    )
)
remDr <- rD$client
remDr$open()
```

Now with our browser ready, we can navigate it to youtube. 


```r
remDr$navigate("https://www.youtube.com")
```

After this step, we should be ready to scrape video metadata. We will scrape a video title, a video uploader, and a video URL. We can get all the videos on the title page with the following CSS selector: `#contents #details #meta`. To get the elements with our selenium browser, we just need to use the `findElements` method.


```r
videoEls  <- remDr$findElements(using = "css", "#contents #details #meta")
```

And now it's time to return the data in the structured form out of the Selenium browser. We can create a small function, that will return `data.frame` with the information we need. 

# Scraping the data


```r
getVideoDetails <- function(videoEl) {
  titleEl <- videoEl$findChildElement("css","#video-title-link")
  metadataEl <- videoEl$findChildElement("css","#metadata")
  
  URL <- titleEl$getElementAttribute("href")[[1]]
  title <- titleEl$getElementText()[[1]]
  uploaderName <- metadataEl$findChildElement("css", "#byline-container")$getElementText()[[1]]
  data.frame(URL = URL, title = title, uploaderName = uploaderName, stringsAsFactors = FALSE)
}
```

As the last step, we will call this function on each element of the videoEls list that we have selected.


```r
details <- lapply(videoEls, getVideoDetails) %>%
  dplyr::bind_rows()
```

# Time to scroll

But what about this dynamic content that is generated when we scroll? We will need to select a page element, that we will use for scrolling. The `body` element is ideal for this, as it wraps the whole web page. After that, we will move to the end of said element. However, as it is infinite scrolling, we will do it just 10 times.


```r
bodyEl <- remDr$findElement("css", "body")
for (i in 1:10) {
  bodyEl$sendKeysToElement(list(key = "end"))
  Sys.sleep(5)
}
```

You might have noticed `Sys.sleep` function in the loop. It is a good practice to wait a while, between doing requests on the website. Here it is especially important, as we need to wait until our webpage loads the new dynamically generated data. After that, we can once again get all the elements using our CSS selector and scrape the details of videos.


```r
videoEls  <- remDr$findElements(using = "css", "#contents #details #meta")
videoDetailsScrolled <- lapply(videoEls  , getVideoDetails) %>%
  dplyr::bind_rows()
```

And that's it. As you can see `videoDetailsScrolled` has more rows than `details` because we have scrolled down 10 times which generated more videos for us to scrape.

I hope that you find this tutorial useful and now know, how to tackle those pesky dynamically generated websites. See you in the next post and scrape responsibly!

