# Scraping financial statements of Slovak financial entities




<p>It is always interesting to go back to your older projects. You can spot, how is your coding style evolving, and how you, as a programmer, are improving. Recently, I had to go through the code of one of my first projects in R and boy, was it a mess. It was supposed to download Financial Statements of all the businesses in Slovakia for a certain year. It worked, barely. But trying to understand the code was a pain. No functions,
random variable names, no documentation… There was even a public API, but for some reason (and I am sure, that at a time It was a “great” one), I have decided not to use it but to scrape web pages directly.</p>
<p>So I have decided this is a great opportunity to see, how would I do things now, and finally do this code some justice and banish this abomination that I have once called code.</p>
<div id="public-api" class="section level1">
<h1>Public API</h1>
<p>First of all, we have to take a look on a public API endpoints and data model. Its full documentation is here (<a href="http://www.registeruz.sk/cruz-public/home/api" class="uri">http://www.registeruz.sk/cruz-public/home/api</a>). From the data model, we are interested in accounting entities and financial statements.
It is also possible to see expected output from the endpoints, which we will use for preparing our database tables.</p>
</div>
<div id="project-structure" class="section level1">
<h1>Project structure</h1>
<p>This time, things should be done right, that’s why I have decided to create R package. This is a first step, that will give this project initial structure. This also gives developer bigger control over which packages are used, which functions we want to export and a lot more. All these great features come at almost no cost, so there is no reason not to create R package. If you haven’t created R package before, Hadley has a great tutorial (<a href="http://r-pkgs.had.co.nz/" class="uri">http://r-pkgs.had.co.nz/</a>) how to do things right.</p>
<p>In the previous project, data were saved as .csv files which were the manually concated. Now we want to have downloaded data stored in a reasonable way. Because of the relational nature of the data, we will use PostgreSQL database. (<a href="https://www.enterprisedb.com/" class="uri">https://www.enterprisedb.com/</a>)</p>
</div>
<div id="preparing-database" class="section level1">
<h1>Preparing database</h1>
<p>After you install your PostgreSQL database, we need to create tables, where we will store our data. All the tables can be created with the following script:</p>
<pre class="sql"><code>CREATE TABLE financial_report_for_financial_statement (
    id serial PRIMARY KEY,
    id_financial_statement VARCHAR (10),
    id_financial_report VARCHAR (10)
);

CREATE TABLE financial_report_title (
    id serial PRIMARY KEY,
    id_financial_report VARCHAR (10),
    nazovUctovnejJednotky VARCHAR (500),
    ico VARCHAR (8),
    dic VARCHAR (10),             
    adresa VARCHAR (500),
    skNace VARCHAR (100),
    typZavierky VARCHAR (30),
    obdobieOd DATE,
    obdobieDo DATE,
    predchadzajuceObdobieOd DATE,
    predchadzajuceObdobieDo DATE,
    datumSchvalenia DATE,
    datumZostavenia DATE,
    oznacenieObchodnehoRegistra VARCHAR (100),
    type VARCHAR (3)
);

CREATE TABLE financial_report_base (
    id serial PRIMARY KEY,
    id_financial_report VARCHAR (10),
    pristupnostDat VARCHAR (20),
    datumPoslednejUpravy DATE,
    zdrojDat VARCHAR(20),
    kodDanovehoUradu VARCHAR(20),
    idSablony VARCHAR(20)
);

DO language &#39;plpgsql&#39;
$$
DECLARE mujAkt text := &#39;CREATE TABLE financial_report_assets_muj(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,46) As i;
DECLARE mujPas text := &#39;CREATE TABLE financial_report_lae_muj(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,44) As i;
DECLARE mujZS text := &#39;CREATE TABLE financial_report_is_muj(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,76) As i;

DECLARE podAkt text := &#39;CREATE TABLE financial_report_assets_pod(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,312) As i;
DECLARE podPas text := &#39;CREATE TABLE financial_report_lae_pod(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,134) As i;
DECLARE podZS text := &#39;CREATE TABLE financial_report_is_pod(id serial PRIMARY KEY, id_financial_report varchar(10),&#39;
    || string_agg(&#39;V&#39; || i::text || &#39; integer&#39;, &#39;,&#39;) || &#39;);&#39;
    FROM generate_series(1,122) As i;


BEGIN
  EXECUTE mujAkt;
    EXECUTE mujPas;
    EXECUTE mujZS;
    
    EXECUTE podAkt;
    EXECUTE podPas;
    EXECUTE podZS;
END;
$$ ;</code></pre>
<p>The script will create all the tables that we will need for the downloaded data. Now, it’s time to prepare code that will fill the tables with data.</p>
</div>
<div id="communication-with-api" class="section level1">
<h1>Communication with API</h1>
<p>Now it is time to prepare funtions, which will communicate with the API. Natural way to communicate with API would be to use different type of HTTP request methods. However this is not the case. If you try <code>httr::GET</code> it will fail, however we can use function <code>readLines</code> from base package. This will return the content of the website which we can then transform into nice object using <code>rjson::fromJSON</code>.</p>
<p>We can create a nice little function for this, as we will use this a lot.</p>
<pre class="r"><code>getObjectFromURL &lt;- function(url) {
  readLines(url, warn = FALSE, encoding = &quot;UTF-8&quot;) %&gt;% 
    rjson::fromJSON()
}</code></pre>
<p>For creating URLs with queries, We could use <code>paste</code> for each each endpoint with all the parameters, however, this solution would be really error prone and could get messy soon. For this we can also use small utility function, which will prepare the URL.</p>
<pre class="r"><code>createUrl &lt;- function(endpoint, ..., baseUrl = &quot;http://www.registeruz.sk/cruz-public/api&quot;) {
  params &lt;- list(...)
  params &lt;- paste(names(params), params, sep = &quot;=&quot;)
  if (length(params) == 0) {
    paste0(baseUrl, endpoint)
  } else {
    paste0(baseUrl, endpoint, &quot;?&quot;, paste0(unlist(params), collapse = &quot;&amp;&quot;))
  }
}</code></pre>
<p>And this last function, which we will use for downloading data is a little bit hacky. While batch downloading, I have noticed, that querying some URLs will throw error, but if we query the same URL next time, it will return the data. So this functions will try to query URL a few times and it will return <code>NULL</code> only if it fails each time.</p>
<pre class="r"><code>tryUntilSuccess &lt;- function(url, numberOfTries = 20, fun = getObjectFromURL) {
  if (numberOfTries == 0) {
    warning(&quot;Unable to read from: &quot;, url)
    NULL
  } else {
    res &lt;- tryCatch(
      fun(url),
      error = function(x) {tryUntilSuccess(url, numberOfTries - 1, fun)}
    )
    res
  }
}
</code></pre>
<p>After we put all this stuff together, we can create nice small functions for communicating with endpoints.</p>
<pre class="r"><code>getChangedFinancialReports &lt;- function(from = &quot;2019-01-01&quot;, maxNumber = 10000, afterId = 1) {
  createUrl(&quot;/uctovne-vykazy&quot;, &quot;zmenene-od&quot; = from, &quot;max-zaznamov&quot; = maxNumber, &quot;pokracovat-za-id&quot; = afterId) %&gt;%
    tryUntilSuccess()
}

getFinancialReportDetails &lt;- function(id) {
  createUrl(&quot;/uctovny-vykaz&quot;, id = id) %&gt;%
    tryUntilSuccess()
}
</code></pre>
</div>
<div id="database-communication" class="section level1">
<h1>Database communication</h1>
<p>Next we need to prepare functions, which will save downloaded data to the database. We pretty much only need database connection function, which will be apending data into the database. Database connection is defined as global variable here as I have wanted to create something similar to singleton. I am not fully satisfied with this solution, as it misses encapsulation, so I will probably look how to approach this stuff in the future.</p>
<pre class="r"><code>DB_CONNECTION &lt;- NULL


getDBConnection &lt;- function() {
  db &lt;- &quot;postgres&quot;
  host_db &lt;- &quot;127.0.0.1&quot;
  db_port &lt;- &quot;5432&quot;
  db_user &lt;- &quot;postgres&quot;
  db_password &lt;- &quot;admin&quot;
  if (is.null(DB_CONNECTION)) {
    message(&quot;Opening new DB connection&quot;)
    DB_CONNECTION &lt;&lt;-
      DBI::dbConnect(
        RPostgres::Postgres(),
        dbname = db,
        host = host_db,
        port = db_port,
        user = db_user,
        password = db_password
      )
  }

  DB_CONNECTION
}


appendIfMissing &lt;- function(x, tableName, con) {
  whereStatement &lt;- lapply(names(x), function(name) {
    glue::glue_sql(&quot;{`name`} NOT IN ({x[[name]]*})&quot;, .con = con)
  }) %&gt;% unlist() %&gt;% glue::glue_collapse(sep = &quot; OR &quot;)

  res &lt;- glue::glue_sql(paste(&quot;SELECT * FROM {`tableName`}&quot;), .con = con) %&gt;%
    RPostgres::dbGetQuery(con, .)
  names(x) &lt;- tolower(names(x))
  toAppend &lt;- dplyr::anti_join(toCharDF(x), toCharDF(res[names(x)]), by = names(x))

  if (nrow(toAppend) &gt; 0) {
    nrOfAppendedRows &lt;- RPostgres::dbAppendTable(con, tableName, toAppend, row.names = NULL)
    usethis::ui_info(paste(&quot;Appended missing values to table:&quot;, nrOfAppendedRows))
  }
}</code></pre>
<p>Except of these functions, that will make our communication with DB easier, we will also need to prepare data to be appended into the database. We are going to split each financial report into multiple different parts. Base part of the financial report, which specifies where the data comes from and which template was used. Then we have title page of the report, which contains informations about financial unit. Final parts that we will extract are assets, liabilities and equity, and income statement of the financial entity.</p>
<p>For these things, we will use following functions:</p>
<pre class="r"><code>getBaseFinReport &lt;- function(finReport) {
  finReport[!names(finReport) %in% c(&quot;prilohy&quot;, &quot;obsah&quot;, &quot;idUctovnejZavierky&quot;, &quot;id&quot;)] %&gt;%
    as.data.frame(stringsAsFactors = FALSE) %&gt;%
    cbind(id_financial_report = finReport$id)
}

getTitleFinReport &lt;- function(finReport) {
  .normalizeDate &lt;- function(x) {
    if (!is.null(x) &amp;&amp; !is.na(x)) addDayToDate(x) else x
  }

  ncol(getAktivaFinReport(finReport))

  title &lt;- finReport$obsah$titulnaStrana
  title$adresa &lt;- paste(title$adresa, collapse = &quot; &quot;)
  if (ncol(getAktivaFinReport(finReport)) == 47) {
    title$type &lt;- &quot;MUJ&quot;
  } else if (ncol(getAktivaFinReport(finReport)) == 313){
    title$type &lt;- &quot;POD&quot;
  }

  title &lt;- title %&gt;% as.data.frame(stringsAsFactors = FALSE)
  title$obdobieOd &lt;- .normalizeDate(title$obdobieOd)
  title$obdobieDo &lt;- .normalizeDate(title$obdobieDo)
  title$predchadzajuceObdobieDo &lt;- .normalizeDate(title$predchadzajuceObdobieDo)
  title$predchadzajuceObdobieOd &lt;- .normalizeDate(title$predchadzajuceObdobieOd)

  cbind(id_financial_report = finReport$id, title)
}

getNumericReportData &lt;- function(finReport, position) {
  finReport$obsah$tabulky[[position]]$data %&gt;%
    as.numeric() %&gt;%
    t() %&gt;%
    as.data.frame() %&gt;%
    cbind(id_financial_report = finReport$id)
}

getAktivaFinReport &lt;- function(finReport) {
  getNumericReportData(finReport, 1)
}

getPasivaFinReport &lt;- function(finReport) {
  getNumericReportData(finReport, 2)
}

getZSFinReport &lt;- function(finReport) {
  getNumericReportData(finReport, 3)
}

addDayToDate &lt;- function(x, day = &quot;01&quot;) {
  paste(x, day, sep = &quot;-&quot;)
}

toCharDF &lt;- function(x) {
  x[] &lt;- lapply(x, as.character)
  x
}
</code></pre>
<p>And this is also pretty much it we only need to combine these functions and we are ready to save financial report into the database with the following function.</p>
<pre class="r"><code>appendFinancialReport &lt;- function(finReport) {
  .appendType &lt;-
    function(x, finReportTitle) {
      if (finReportTitle$type == &quot;MUJ&quot;)
        paste0(x, &quot;_muj&quot;)
      else
        paste0(x, &quot;_pod&quot;)
    }
  message(&quot;Process report:&quot;, finReport$id)

  con &lt;- getDBConnection()
  finReportBase &lt;- getBaseFinReport(finReport)
  finReportTitle &lt;- getTitleFinReport(finReport)
  if (!is.null(finReportTitle$type)) {

    finReportsForFinStatement &lt;- data.frame(
      id_financial_report = finReport$id,
      id_financial_statement = finReport$idUctovnejZavierky,
      stringsAsFactors = FALSE
    )
  
    message(&quot;Add report with id:&quot;, finReport$id)
      tryCatch(
        {
          RPostgres::dbBegin(con)
          appendIfMissing(finReportBase, &quot;financial_report_base&quot;, con)
          appendIfMissing(finReportTitle, &quot;financial_report_title&quot;, con)
          appendIfMissing(finReportsForFinStatement, &quot;financial_report_for_financial_statement&quot;, con)
          appendIfMissing(getAktivaFinReport(finReport), .appendType(&quot;financial_report_assets&quot;, finReportTitle), con)
          appendIfMissing(getPasivaFinReport(finReport), .appendType(&quot;financial_report_lae&quot;, finReportTitle), con)
          appendIfMissing(getZSFinReport(finReport), .appendType(&quot;financial_report_is&quot;, finReportTitle), con)
          RPostgres::dbCommit(con)
        },
        error = function(e) {RPostgres::dbRollback(con)}
      )
  }
}</code></pre>
<p>As you can see, we have appending to the database wrapped in the <code>tryCatch</code> function. We do this, to keep transaction atomic. This means, that we will commit all the data into the database or nothing will occurs. So in case of the error, <code>dbRollback</code> is called and the transaction is reverted.</p>
</div>
<div id="putting-it-all-together" class="section level1">
<h1>Putting it all together</h1>
<p>Now it is time to combine everything we have prepared, and start scraping data. Thanks to our modular design, this should be pretty easy. All we have to do now is to get all the IDs of financial reports, that has changed since certain date and then download each of them and save it into the database.</p>
<pre class="r"><code>getAllChangedFinancialReports(&quot;2020-05-01&quot;) %&gt;%
  .$result.id %&gt;%
  lapply(function(x) {
    getFinancialReportDetails(x) %&gt;%
      appendFinancialReport()
  })
</code></pre>
<p>This should download all the financial report data which we can further analyse.</p>
</div>

