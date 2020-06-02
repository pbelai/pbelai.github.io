# Scraping financial statements of Slovak financial entities




<p>It is always interesting to go back to your older projects. You can spot, how is your coding style evolving, and how you, as a programmer, are improving. Recently, I had to go through the code of one of my first projects in R and boy, was it a mess. It was supposed to download Financial Statements of all the businesses in Slovakia for a certain year. It worked, barely. But trying to understand the code was a pain. No functions,
random variable names, no documentation… There was even a public API, but for some reason (and I am sure, that at a time It was a “great” one), I have decided not to use it but to scrape web pages directly.</p>
<p>So I have decided this is a great opportunity to see, how would I do things now, and finally do this code some justice and banish this abomination that I have once called code.</p>
<div id="public-api" class="section level3">
<h3>Public API</h3>
<p>First of all, we have to take a look on a public API endpoints and data model. Its full documentation is here (<a href="http://www.registeruz.sk/cruz-public/home/api" class="uri">http://www.registeruz.sk/cruz-public/home/api</a>). From the data model, we are interested in accounting entities and financial statements.
It is also possible to see expected output from the endpoints, which we will use for preparing our database tables.</p>
</div>
<div id="project-structure" class="section level3">
<h3>Project structure</h3>
<p>This time, things should be done right, that’s why I have decided to create R package. This is a first step, that will give this project initial structure. This also gives developer bigger control over which packages are used, which functions we want to export and a lot more. All these great features come at almost no cost, so there is no reason not to create R package. If you haven’t created R package before, Hadley has a great tutorial (<a href="http://r-pkgs.had.co.nz/" class="uri">http://r-pkgs.had.co.nz/</a>) how to do things right.</p>
<p>In the previous project, data were saved as .csv files which were the manually concated. Now we want to have downloaded data stored in a reasonable way. Because of the relational nature of the data, we will use PostgreSQL database. (<a href="https://www.enterprisedb.com/" class="uri">https://www.enterprisedb.com/</a>) For connecting to the database we will use THIS package.</p>
<p>It is possible to see that we will have to separate this will have to address two separate concerns:</p>
<ul>
<li>Communication with API</li>
<li>Database communication</li>
<li>Scraping the data</li>
</ul>
<p>After you have created new R package and have installed PostgreSQL, we can finally start scraping the data.</p>
</div>
<div id="preparing-database" class="section level3">
<h3>Preparing database</h3>
<pre class="sql"><code>CREATE TABLE accounting_entity (
    id VARCHAR (10) PRIMARY KEY,  --! EJ
    obdobieOd DATE,
    obdobieDo DATE,
    datumPodania DATE,
    datumZostavenia DATE,
    datumSchvalenia DATE,
    datumZostaveniaK DATE,
    datumPrilozeniaSpravyAuditora DATE,
    nazovUJ VARCHAR (500),
    ico CHAR (8),
    dic CHAR (10),
    nazovFondu VARCHAR (500),
    leiKod VARCHAR (20),
    idUJ VARCHAR (10),
    konsolidovana BOOLEAN,
    konsolidovanaZavierkaUstrednejStatnejSpravy BOOLEAN,
    suhrnnaUctovnaZavierkaVerejnejSpravy BOOLEAN,
    typ VARCHAR (15),
    idUctovnychVykazov VARCHAR (10),
    zdrojDat VARCHAR (30),
    datumPoslednejUpravy DATE 
);


CREATE TABLE accounting_entity (
    id VARCHAR (10) PRIMARY KEY,  --! EJ
    ico CHAR (8),
    dic CHAR (10),
    sid VARCHAR (5),
    nazovUJ VARCHAR (500),
    mesto VARCHAR (200),
    ulica VARCHAR (200),
    psc VARCHAR (10),
    datumZalozenia DATE,
    datumZrusenia DATE,
    pravnaForma VARCHAR (100),
    skNace VARCHAR (100),
    velkostOrganizacie VARCHAR (100),
    druhVlastnictva VARCHAR (100),
    kraj VARCHAR (100),
    okres VARCHAR (100),
    sidlo VARCHAR (100),
    konsolidovana bool,
    idUctovnychZavierok VARCHAR (10),
    idVyrocnychSprav VARCHAR (10),
    zdrojDat VARCHAR (30),
    datumPoslednejUpravy DATE 
);</code></pre>
</div>
<div id="communication-with-api" class="section level3">
<h3>Communication with API</h3>
</div>
<div id="database-communication" class="section level3">
<h3>Database communication</h3>
</div>
<div id="putting-it-all-together" class="section level3">
<h3>Putting it all together</h3>
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
<p><img src="/posts/2020-06-01-scraping-financial-statements-of-slovak-financial-entities.en_files/figure-html/pressure-1.png" width="672" /></p>
<p>Note that the <code>echo = FALSE</code> parameter was added to the code chunk to prevent printing of the R code that generated the plot.</p>
</div>
