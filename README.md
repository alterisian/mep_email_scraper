---
title: "List of e-mail addresses of MEPs"
author: "Roman Luštrik"
date: "06 marec 2016"
output: html_document
---

The following script will try to scape a list of MEPs for their e-mails found at [this address](http://www.europarl.europa.eu/meps/en/full-list.html?filter=all&leg=).

For this method to work, in addition to what is loaded in the script below, you will need to have [PhantomJS](http://phantomjs.org/) installed and in your PATH or somehow visible to the `system()` call.

This list was scraped in March 2016 and the file is available in the repository `<> Code`. If you need to update it, feel free to scrape it again.

Special thanks to [m0nhawk](http://stackoverflow.com/users/1030110/m0nhawk) and [Jaap](http://stackoverflow.com/users/2204410/jaap) for tips on webscraping.

```r
library(rvest)

# from https://www.datacamp.com/community/tutorials/scraping-javascript-generated-data-with-r
writeJS <- function(link, file, html.name) {
  writeLines(sprintf("
var webPage = require('webpage');
var page = webPage.create();

var fs = require('fs');
var path = '%s'

page.open('%s', function (status) {
  var content = page.content;
  fs.write(path,content,'w')
  phantom.exit();
});
", html.name, link),
             con = file)
  system(sprintf("phantomjs %s --local-storage-path='%s'", file, html.name))
}

# get a list of all MEPs
writeJS(link = "http://www.europarl.europa.eu/meps/en/full-list.html?filter=all&leg=",
        file = "./js/meps.js", html.name = "./js/meps.html")

meps <- read_html("./js/meps.html")

mep.links <- meps %>%
  html_nodes("div#content_left div.zone_info_mep a") %>%
  html_attr("href") %>%
  unique() %>%
  paste("http://www.europarl.europa.eu", ., sep = "")

# construct names of scrabed files and results
xy <- data.frame(
  jsname = paste("./js/", sub(".html", replacement = ".js", x = unlist(lapply(strsplit(mep.links, "/"), "[", 7))), sep = ""),
  html.name = paste("./html/", unlist(lapply(strsplit(mep.links, "/"), "[", 7)), sep = ""),
  link = mep.links
)

for (i in 1:nrow(xy)) {
  writeJS(link = as.character(xy[i, "link"]),
          file = as.character(xy[i, "jsname"]), html.name = as.character(xy[i, "html.name"]))
  epname <- read_html(as.character(xy[i, "html.name"]))
  epname <- epname %>% html_nodes("div#content_right a") %>% html_attr("href")
  write(sub("mailto:", replacement = "", x = epname[grepl("mailto", epname)]), file = "meps.txt", append = TRUE)
  sleep <- rnorm(1, mean = 3, sd = 1)
  if (sleep <= 0) sleep <- 1
  Sys.sleep(sleep) # sleep a bit, although downloading of a website already takes long
}
```

