---
title: "List of e-mail addresses of MEPs"
author: "Roman Luštrik"
date: "06 marec 2016"
output: html_document
---

The following script will try to scape a list of MEPs for their e-mails found at [this address](http://www.europarl.europa.eu/meps/en/full-list.html?filter=all&leg=). [This](https://github.com/romunov/mep_email_scraper/blob/master/meps.txt) is the result, with an e-mail friendy version which can be found [here](https://github.com/romunov/mep_email_scraper/blob/master/email_friendly.txt). Source code can be [found here](https://github.com/romunov/mep_email_scraper/blob/master/source_code_mep_scraper.R).

For this method to work, in addition to what is loaded in the script below, you will need to have [PhantomJS](http://phantomjs.org/) installed and in your PATH or somehow visible to the `system()` call.

This list was scraped in March 2016 and the file is available in the repository `<> Code`. If you need to update it, feel free to scrape it again.

NOTE: Some members do not have an email listed and others have more than one, which is why there are more than 751 contacts.

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
        file = "./js/meps.js", html.name = "./html/meps.html")

meps <- read_html("./html/meps.html")

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
  # download page
  writeJS(link = as.character(xy[i, "link"]),
          file = as.character(xy[i, "jsname"]), html.name = as.character(xy[i, "html.name"]))
  
  # read data (name, email)
  epname <- read_html(as.character(xy[i, "html.name"]))
  mep.name <- epname %>% html_nodes("a#mep_name_button") %>% html_attr("title")
  epname <- epname %>% html_nodes("div#content_right a") %>% html_attr("href")
  email <- sub("mailto:", replacement = "", x = epname[grepl("mailto", epname)])
  
  if (identical(email, character(0))) email <- "(no email)"
  
  towrite <- data.frame(email = email, name = mep.name)
  write.table(towrite, file = "meps.txt", append = TRUE, row.names = FALSE,
              col.names = FALSE, quote = FALSE, sep = "\t")
  
  # sleep
  sleep <- rpois(1, lambda = 4)
  if (sleep <= 0) sleep <- 1
  Sys.sleep(sleep) # sleep a bit, although downloading of a website already takes long
}

# Write emails in a format that is easy to copy/paste into your favorite email client.
# Note that there may be a limit of how many addresses the client can process.
meps.email <- read.table("meps.txt", header = FALSE, sep = "\t")

# remove members with no e-mail
meps.email <- meps.email[!(meps.email$V1 %in% c("(no email)")), ]
write(paste(meps.email$V1, collapse = ", "), file = "email_friendly.txt")
```

