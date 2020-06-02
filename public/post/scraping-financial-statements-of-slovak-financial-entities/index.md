# Scraping financial statements of Slovak financial entities




<p>It is always interesting to go back to your older projects. You can spot, how is your coding style evolving, and how you, as a programmer, are improving. Recently, I had to go through the code of one of my first projects in R and boy, was it a mess. It was supposed to download Financial Statements of all the businesses in Slovakia for a certain year. It worked, barely. But trying to understand the code was a pain. No functions,
random variable names, no documentation… There was even a public API, but for some reason (and I am sure, that at a time It was a “great” one), I have decided not to use it but to scrape web pages directly.</p>
<p>So I have decided this is a great opportunity to see, how would I do things now, and finally do this code some justice and banish this abomination that I have once called code.</p>
<div id="public-api" class="section level3">
<h3>Public API</h3>
<p>First of all, we have to take a look on a public API endpoints and data model. Its full documentation is here (<a href="http://www.registeruz.sk/cruz-public/home/api" class="uri">http://www.registeruz.sk/cruz-public/home/api</a>). From the data model, we are interested in accounting entities and financial statements.
As you can see from the</p>
</div>
<div id="code-structure" class="section level3">
<h3>Code structure</h3>
<p>This time, things should be done right, that’s why I have decided to create R package. This is a first step, that will give this project initial structure. This also gives developer bigger control over which packages are used, which functions we want to export and lot more. All these great features come at almost no cost, so there is no reason not to create R package. If you haven’t created R package before, Hadley has a great tutorial (<a href="http://r-pkgs.had.co.nz/" class="uri">http://r-pkgs.had.co.nz/</a>) how to do things right.</p>
<p>This is an R Markdown document. Markdown is a simple formatting syntax for authoring HTML, PDF, and MS Word documents. For more details on using R Markdown see <a href="http://rmarkdown.rstudio.com" class="uri">http://rmarkdown.rstudio.com</a>.</p>
<p>When you click the <strong>Knit</strong> button a document will be generated that includes both content as well as the output of any embedded R code chunks within the document. You can embed an R code chunk like this:</p>
<pre class="r"><code>summary(cars)
##      speed           dist       
##  Min.   : 4.0   Min.   :  2.00  
##  1st Qu.:12.0   1st Qu.: 26.00  
##  Median :15.0   Median : 36.00  
##  Mean   :15.4   Mean   : 42.98  
##  3rd Qu.:19.0   3rd Qu.: 56.00  
##  Max.   :25.0   Max.   :120.00</code></pre>
</div>
<div id="including-plots" class="section level2">
<h2>Including Plots</h2>
<p>You can also embed plots, for example:</p>
<p><img src="/post/2020-06-01-scraping-financial-statements-of-slovak-financial-entities.en_files/figure-html/pressure-1.png" width="672" /></p>
<p>Note that the <code>echo = FALSE</code> parameter was added to the code chunk to prevent printing of the R code that generated the plot.</p>
</div>
