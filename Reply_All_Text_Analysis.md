Reply All Text Analysis
================
Noah Landesberg
3/17/2018

**Download CSV here:
<https://github.com/landesbergn/reply-all/blob/master/reply_all_text_data.csv>**

One of my favorite podcasts is [Reply
All](https://www.gimletmedia.com/reply-all), a show (roughly) about
technology and the internet. It is hosted by PJ Vogt and Alex Goldman,
as well as some fantastic reporters and producers who contribute
regularly, and tells some of the most fascinating stories about the way
we interact with technology.

The show has been in production since 2014, and for a time felt like a
great little secret, but their website now indicates that the show is
downloaded “around 3.5 million times per month.” If you haven’t
listened, I’d highly recommend checking it out. Some of my favorite
episodes are (in no particular order):

  - [The
    Cathedral](https://www.gimletmedia.com/reply-all/50-the-cathedral#episode-player)  
  - [Boy in
    Photo](https://www.gimletmedia.com/reply-all/79-boy-in-photo#episode-player)  
  - [The Grand Tapestry of
    Pepe](https://www.gimletmedia.com/reply-all/77-the-grand-tapestry-of-pepe#episode-player)  
  - [Shine on You Crazy
    Goldman](https://www.gimletmedia.com/reply-all/44-shine-on-you-crazy-goldman#episode-player)  
  - [Long Distance
    pt.1](https://www.gimletmedia.com/reply-all/long-distance#episode-player)
    and [Long Distance
    pt.2](https://www.gimletmedia.com/reply-all/103-long-distance-part-ii#episode-player)  
  - [Zardulu](https://www.gimletmedia.com/reply-all/zardulu#episode-player)

I thought it would be a fun project to take the transcripts from every
episode of Reply All and see what we can learn about the show. As is
often the case in data science, 80% of the challenge is to gather and
clean the data.

To gather data, we will make use of a few R packages. `rvest` will do
most of the web scraping, and `dplyr`, `tidyr`, and `stringr` will help
to clean and organize our data. So, without further ado, let’s load ’em
up\!

``` r
library(rvest) # for web scraping
library(dplyr) # for data tidying
library(stringr) # for working with strings
library(tidytext) # analyze text data!
library(ggplot2) # make nice plots
library(scales) # make plots even nicer
library(tidyr) # for data organization
library(purrr) # for iteration
```

To get the transcripts of all the episodes, first we need to know which
episodes are out there. By navigating to the [Reply All
website](https://www.gimletmedia.com/reply-all/all#all-episodes-list),
we can load a handy list of all episodes as well as some hyperlinks that
will take us into the individual web page for each episode. This link
will serve as the starting point for our
web-scraping.

``` r
reply_all_url <- "https://www.gimletmedia.com/reply-all/all#all-episodes-list"
```

We will use some functions from `rvest` to pull the hyperlinks to every
episode we will eventually want to read the transcript from. This was my
first time using `rvest`, and I found [this
post](https://stat4701.github.io/edav/2015/04/02/rvest_tutorial/) from
Justin Law and Jordan Rosenblum to be helpful, as well as [this RStudio
blog
post](https://blog.rstudio.com/2014/11/24/rvest-easy-web-scraping-with-r/)
from Hadley Wickham. To get the episode links, we will look for all HTML
in the `<a>` anchor tag and pull the `href` attribute where the
hyperlink lives.

``` r
episode_links <- reply_all_url %>%
  read_html() %>% # read HTML from page
  html_nodes("a") %>% # look for 'anchor' html tag <a>
  html_attr("href") %>% # get the href attribute from the tag, which is where the hyperlink is stored
  tibble(link = .) # put the data in a tibble with the variable episode

tail(episode_links) %>% knitr::kable()
```

| link                                                                   |
| :--------------------------------------------------------------------- |
| /reply-all/1-an-app-sends-a-stranger-to-say-i-love-you\#episode-player |
| /                                                                      |
| /about                                                                 |
| /careers                                                               |
| /terms-of-service                                                      |
| /privacy-policy                                                        |

We can tell by looking at the `head` (first 6 rows) and the `tail` (last
6 rows) of the list of episode links that there are some links to other
pages here that are not of interest, like the privacy policy or the
terms of service.

Let’s clean this up a little bit, by only including episodes that
reference `#episode-player` at the end of their link, indicated that
there is an episode there, and not some other webpage. We can also
filter out episodes that are rebroadcasts or presentations of other
shows in the Reply All feed. The process of filtering down this list
took some trial and error, as the formatting of episode titles in the
hyperlink was inconsistent.

``` r
episode_links <- episode_links %>%
  filter(
    str_detect(link, "#episode-player"), # must link to an epsiode player
    !str_detect(link, "re-broadcast"), # no re-broadcasts
    !str_detect(link, "rebroadcast"), # or rebroadcasts
    !str_detect(link, "revisited"), # or revisits
    !str_detect(link, "-2#episode-player"), # or other rebroadcasts
    !str_detect(link, "presents"), # or presentations of other shows
    !str_detect(link, "introducing"), # or introductions of other shows
    !str_detect(link, "updated"), # or updated versions of old episodes
    link != "/reply-all/6-this-proves-everything#episode-player", # sneaky update of an old episode that is not labeled well
    link != "/reply-all/104-the-case-of-the-phantom-caller#episode-player" # another one
  ) %>%
  distinct() # and after all of that, no duplicates

episode_links <- episode_links %>% 
  mutate(episode_number = nrow(episode_links) + 1 - row_number()) # add in the episode number

head(episode_links) %>% knitr::kable()
```

| link                                                                 | episode\_number |
| :------------------------------------------------------------------- | --------------: |
| /reply-all/135-the-robocall-conundrum\#episode-player                |             135 |
| /reply-all/134-the-year-of-the-wallop\#episode-player                |             134 |
| /reply-all/133-reply-alls-2018-year-end-extravaganza\#episode-player |             133 |
| /reply-all/132-negative-mount-pleasant\#episode-player               |             132 |
| /reply-all/131-surefire-investigations\#episode-player               |             131 |
| /reply-all/130-lizard\#episode-player                                |             130 |

We can also add in the episode number, which I originally tried to parse
out from the string, but I am not good at regular expression, so I just
used some math to back in to it.

Finally, we add in the full web link, by appending the hyperlink
extension to the Gimlet Media homepage (`gimetmedia.com`).

``` r
ep_data <- episode_links %>%
  mutate(
    full_link = paste0("https://www.gimletmedia.com", link)
  )

head(ep_data) %>% knitr::kable()
```

| link                                                                 | episode\_number | full\_link                                                                                       |
| :------------------------------------------------------------------- | --------------: | :----------------------------------------------------------------------------------------------- |
| /reply-all/135-the-robocall-conundrum\#episode-player                |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player>                |
| /reply-all/134-the-year-of-the-wallop\#episode-player                |             134 | <https://www.gimletmedia.com/reply-all/134-the-year-of-the-wallop#episode-player>                |
| /reply-all/133-reply-alls-2018-year-end-extravaganza\#episode-player |             133 | <https://www.gimletmedia.com/reply-all/133-reply-alls-2018-year-end-extravaganza#episode-player> |
| /reply-all/132-negative-mount-pleasant\#episode-player               |             132 | <https://www.gimletmedia.com/reply-all/132-negative-mount-pleasant#episode-player>               |
| /reply-all/131-surefire-investigations\#episode-player               |             131 | <https://www.gimletmedia.com/reply-all/131-surefire-investigations#episode-player>               |
| /reply-all/130-lizard\#episode-player                                |             130 | <https://www.gimletmedia.com/reply-all/130-lizard#episode-player>                                |

Now that we have a link to every episode we want to parse, it is time to
write a function that, when given a link, returns a transcript. This
took a lot of time to nail down. As alluded to, the data and transcript
formatting as inconsistent and messy. It took a fair amount of time and
iteration to try to get this right. I a lot of time on
<https://regex101.com> testing out stuff.

``` r
getTranscript <- function(episode_link) {
  
  # print statement helps to understand the progress of the function when it is running (commented out for now)
  # print(episode_link)

  # get the transcript from the webpage by referencing the CSS node '.episode-transcript'
  transcript <- episode_link %>%
    read_html() %>%
    html_nodes(".episode-transcript") %>%
    html_text()
  
  # this is an attempt at spliting up the text by speaker, so that
  # each string is seperated by the person who said it.
  episode_text <- tibble(
    unlist(strsplit(transcript, "[^.a-z]+:", perl = T)) # this regex is to identify [ALEX]: or [PJ]: at the begining of a line
  ) %>% setNames("text")
  
  episode_text <- episode_text %>% 
    mutate(
      text = trimws(text) # remove leading and trailing white-space
    ) %>%
    filter(
      tolower(text) != "transcript\n        ", # get rid of line indicating start of transcript
      tolower(text) != "transcript", # or this type of line indicating the start of transcript,
      tolower(text) != "[theme music", # get rid of Theme (or theme music)
      tolower(text) != "[intro music", # get rid of Intro Music (or intro music)
      tolower(text) != "transcript\n        [intro music", # or a mix of things
      tolower(text) != "transcript\n        [theme music", # or a different mix of things
      tolower(text) != "transcript\n        [reply all theme"
    )

  # ok this is gross, but handling some specific episodes where the transcript is not entered in a consistant way
  if (episode_link == "https://www.gimletmedia.com/reply-all/79-boy-in-photo#episode-player") {
    speaker <- rbind(
      "PJ", 
      data.frame(gsubfn::strapply(transcript, "[^.a-z]+:", c, perl = TRUE), stringsAsFactors = FALSE) %>% setNames("speaker")
    )
  } else if (episode_link == "https://www.gimletmedia.com/reply-all/52-raising-the-bar#episode-player") {
    speaker <- rbind(
      "PJ", 
      data.frame(gsubfn::strapply(transcript, "[^.a-z]+:", c, perl = TRUE), stringsAsFactors = FALSE) %>% setNames("speaker")
    )    
    episode_text <- episode_text %>% mutate(text = str_replace_all(text, "Transcript\n        PJ Vogt: ", ""))
  } else if (episode_link == "https://www.gimletmedia.com/reply-all/31-bonus-the-reddit-implosion-explainer#episode-player") {
    speaker <- rbind(
      "PJ", 
      data.frame(gsubfn::strapply(transcript, "[^.a-z]+:", c, perl = TRUE), stringsAsFactors = FALSE) %>% setNames("speaker")
    )
  } else if (episode_link == "https://www.gimletmedia.com/reply-all/2-instagram-for-doctors#episode-player") {
    speaker <- rbind(
      "PJ", 
      data.frame(gsubfn::strapply(transcript, "[^.a-z]+:", c, perl = TRUE), stringsAsFactors = FALSE) %>% setNames("speaker")
    )
    episode_text <- episode_text %>% mutate(text = str_replace_all(text, fixed("Transcript\n        [THEME SONG]PJ Vogt: "), ""))
  } else {
    speaker <- gsubfn::strapply(transcript, "[^.a-z]+:", c, perl = TRUE) %>% setNames("speaker")
  }

  transcript_clean <- data.frame(episode_text, speaker) %>%
    mutate(
      speaker = trimws(speaker),
      speaker = str_replace_all(speaker, "[^A-Z ]", ""),
      speaker = str_replace_all(speaker, "THEME MUSIC", ""),
      speaker = str_replace_all(speaker, "RING", ""),
      speaker = str_replace_all(speaker, "MUSIC", ""),
      speaker = str_replace_all(speaker, "BREAK", "")
    )

  # do some light cleaning of the transcript
  transcript_new <- transcript_clean %>%
    mutate(
      linenumber = row_number() # get the line number (1 is the first line, 2 is the second, etc.)
    ) %>%
    select(speaker, text, linenumber)

  # convert normal boring text into exciting cool tidy text
  transcript_tidy <- transcript_new %>%
    unnest_tokens(word, text)

  return(transcript_tidy)
}
```

Now that we have a function, let’s test it for a single episode to show
it in
action.

``` r
getTranscript("https://www.gimletmedia.com/reply-all/114-apocalypse-soon#episode-player") %>% 
  head() %>% 
  knitr::kable()
```

|     | speaker | linenumber | word      |
| --- | :------ | ---------: | :-------- |
| 1   | PJ VOGT |          1 | hey       |
| 1.1 | PJ VOGT |          1 | everybody |
| 1.2 | PJ VOGT |          1 | we        |
| 1.3 | PJ VOGT |          1 | are       |
| 1.4 | PJ VOGT |          1 | back      |
| 1.5 | PJ VOGT |          1 | it’s      |

Now we can use `purrr` to iterate through every episode and ‘map’ the
function `getTranscript` to each episode link. I learned a lot about
iterating with `purrr` from [this
tutorial](https://jennybc.github.io/purrr-tutorial/) from Jenny Brian
and the chapter from [R for Data Science on
Iteration](http://r4ds.had.co.nz/iteration.html) by Garrett Grolemund
and Hadley Wickham. This takes ~3 minutes to run, depending on your
internet
connection.

``` r
# use purrr to map the 'getTranscript' function over all of the URLs in the ep_data data frame
ep_data <- ep_data %>%
  mutate(
    transcript = map(full_link, getTranscript)
  )

# unnest the resuls into one big data frame
tidy_ep_data <- ep_data %>%
  unnest(transcript)

head(tidy_ep_data) %>% knitr::kable()
```

| link                                                  | episode\_number | full\_link                                                                        | speaker | linenumber | word   |
| :---------------------------------------------------- | --------------: | :-------------------------------------------------------------------------------- | :------ | ---------: | :----- |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | from   |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | gimlet |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | this   |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | is     |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | reply  |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | all    |

Now that we have a big data frame, we can do a little more cleaning of
the data. Arguably, this is avoidable with more intelligent regex and
string work earlier, but this cleanup will have to do for now. I briefly
use the `zoo` package to fill in some NA values in the `speaker` column
using the previous non-NA value (inspired by [this StackoverFlow
answer](https://stackoverflow.com/a/7735681/9185424)).

``` r
# turn missing values to NA and then fill using 
# the na.locf (last observation carried forward) function from the 'zoo' package
tidy_ep_data <- tidy_ep_data %>% 
  mutate(
    speaker = if_else(speaker == "", NA_character_, speaker),
    speaker = zoo::na.locf(speaker)
  )

# get the list of speakers clean
tidy_ep_data_clean <- tidy_ep_data %>%
  filter(
    !grepl("CREDIT", speaker), # remove credit chit-chat
    !grepl("AD ", speaker), # remove ad chit-chat
    !grepl("THEME", speaker), # remove theme chit-chat
    speaker != "ADPJ",
    speaker != "ADALEX",
    speaker != "OUTPJ",
    speaker != "OUTALEX"
  ) %>% 
  mutate(
    speaker = trimws(speaker),
    speaker = case_when(
      speaker == "ALEX" ~ "ALEX GOLDMAN",
      speaker == "REPLY ALL ALEX GOLDMAN" ~ "ALEX GOLDMAN",
      speaker == "GOLDMAN" ~ "ALEX GOLDMAN",
      speaker == "AG" ~ "ALEX GOLDMAN",
      speaker == "PJ" ~ "PJ VOGT",
      speaker == "REPLY ALL PJ VOGT" ~ "PJ VOGT",
      speaker == "BLUMBERG" ~ "ALEX BLUMBERG",
      speaker == "AB" ~ "ALEX BLUMBERG",
      speaker == "SRUTHI" ~ "SRUTHI PINNAMANENI",
      TRUE ~ speaker
    )
  )
```

And after all of that, we now have some sort of nice text data from
every episode of Reply All\!

``` r
glimpse(tidy_ep_data_clean)
```

    ## Observations: 704,321
    ## Variables: 6
    ## $ link           <chr> "/reply-all/135-the-robocall-conundrum#episode-pl…
    ## $ episode_number <dbl> 135, 135, 135, 135, 135, 135, 135, 135, 135, 135,…
    ## $ full_link      <chr> "https://www.gimletmedia.com/reply-all/135-the-ro…
    ## $ speaker        <chr> "PJ VOGT", "PJ VOGT", "PJ VOGT", "PJ VOGT", "PJ V…
    ## $ linenumber     <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3…
    ## $ word           <chr> "from", "gimlet", "this", "is", "reply", "all", "…

``` r
head(tidy_ep_data_clean) %>% knitr::kable()
```

| link                                                  | episode\_number | full\_link                                                                        | speaker | linenumber | word   |
| :---------------------------------------------------- | --------------: | :-------------------------------------------------------------------------------- | :------ | ---------: | :----- |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | from   |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | gimlet |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | this   |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | is     |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | reply  |
| /reply-all/135-the-robocall-conundrum\#episode-player |             135 | <https://www.gimletmedia.com/reply-all/135-the-robocall-conundrum#episode-player> | PJ VOGT |          1 | all    |

We can write the data to a `.csv` for anyone to use in the future.

``` r
readr::write_csv(tidy_ep_data_clean, "reply_all_text_data.csv")
```

In part 2, we will start to analyze this data\!
