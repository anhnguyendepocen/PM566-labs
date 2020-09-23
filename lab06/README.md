Lab 06 - Text Mining
================

``` r
knitr::opts_chunk$set(eval = TRUE, include  = TRUE)
```

# Learning goals

  - Use `unnest_tokens()` and `unnest_ngrams()` to extract tokens and
    ngrams from text.
  - Use dplyr and ggplot2 to analyze text data

# Lab description

For this lab we will be working with a new dataset. The dataset contains
transcription samples from <https://www.mtsamples.com/>. And is loaded
and “fairly” cleaned at
<https://raw.githubusercontent.com/USCbiostats/data-science-data/master/00_mtsamples/mtsamples.csv>.

This markdown document should be rendered using `github_document`
document.

# Setup the Git project and the GitHub repository

1.  Go to your documents (or wherever you are planning to store the
    data) in your computer, and create a folder for this project, for
    example, “PM566-labs”

2.  In that folder, save [this
    template](https://raw.githubusercontent.com/USCbiostats/PM566/master/content/assignment/06-lab.Rmd)
    as “README.Rmd”. This will be the markdown file where all the magic
    will happen.

3.  Go to your GitHub account and create a new repository, hopefully of
    the same name that this folder has, i.e., “PM566-labs”.

4.  Initialize the Git project, add the “README.Rmd” file, and make your
    first commit.

5.  Add the repo you just created on GitHub.com to the list of remotes,
    and push your commit to origin while setting the upstream.

### Setup packages

You should load in `dplyr`, (or `data.table` if you want to work that
way), `ggplot2` and `tidytext`. If you don’t already have `tidytext`
then you can install with

``` r
install.packages("tidytext")
```

### read in Medical Transcriptions

Loading in reference transcription samples from
<https://www.mtsamples.com/>

``` r
library(tidyverse)
library(tidytext)
mt_samples <- read_csv("https://raw.githubusercontent.com/USCbiostats/data-science-data/master/00_mtsamples/mtsamples.csv")
mt_samples <- mt_samples %>%
  select(description, medical_specialty, transcription)

head(mt_samples)
```

    ## # A tibble: 6 x 3
    ##   description                  medical_specialty   transcription                
    ##   <chr>                        <chr>               <chr>                        
    ## 1 A 23-year-old white female … Allergy / Immunolo… "SUBJECTIVE:,  This 23-year-…
    ## 2 Consult for laparoscopic ga… Bariatrics          "PAST MEDICAL HISTORY:, He h…
    ## 3 Consult for laparoscopic ga… Bariatrics          "HISTORY OF PRESENT ILLNESS:…
    ## 4 2-D M-Mode. Doppler.         Cardiovascular / P… "2-D M-MODE: , ,1.  Left atr…
    ## 5 2-D Echocardiogram           Cardiovascular / P… "1.  The left ventricular ca…
    ## 6 Morbid obesity.  Laparoscop… Bariatrics          "PREOPERATIVE DIAGNOSIS: , M…

``` r
mt_samples$transcription[1]
```

    ## [1] "SUBJECTIVE:,  This 23-year-old white female presents with complaint of allergies.  She used to have allergies when she lived in Seattle but she thinks they are worse here.  In the past, she has tried Claritin, and Zyrtec.  Both worked for short time but then seemed to lose effectiveness.  She has used Allegra also.  She used that last summer and she began using it again two weeks ago.  It does not appear to be working very well.  She has used over-the-counter sprays but no prescription nasal sprays.  She does have asthma but doest not require daily medication for this and does not think it is flaring up.,MEDICATIONS: , Her only medication currently is Ortho Tri-Cyclen and the Allegra.,ALLERGIES: , She has no known medicine allergies.,OBJECTIVE:,Vitals:  Weight was 130 pounds and blood pressure 124/78.,HEENT:  Her throat was mildly erythematous without exudate.  Nasal mucosa was erythematous and swollen.  Only clear drainage was seen.  TMs were clear.,Neck:  Supple without adenopathy.,Lungs:  Clear.,ASSESSMENT:,  Allergic rhinitis.,PLAN:,1.  She will try Zyrtec instead of Allegra again.  Another option will be to use loratadine.  She does not think she has prescription coverage so that might be cheaper.,2.  Samples of Nasonex two sprays in each nostril given for three weeks.  A prescription was written as well."

``` r
names(mt_samples)
```

    ## [1] "description"       "medical_specialty" "transcription"

-----

## Question 1: What specialties do we have?

We can use `count()` from `dplyr` to figure out how many different
catagories do we have? Are these catagories related? overlapping? evenly
distributed?

``` r
mt_samples %>%
  count(medical_specialty, sort = TRUE)
```

    ## # A tibble: 40 x 2
    ##    medical_specialty                 n
    ##    <chr>                         <int>
    ##  1 Surgery                        1103
    ##  2 Consult - History and Phy.      516
    ##  3 Cardiovascular / Pulmonary      372
    ##  4 Orthopedic                      355
    ##  5 Radiology                       273
    ##  6 General Medicine                259
    ##  7 Gastroenterology                230
    ##  8 Neurology                       223
    ##  9 SOAP / Chart / Progress Notes   166
    ## 10 Obstetrics / Gynecology         160
    ## # … with 30 more rows

  - there are 40 categories

  - the specialty with the largest count (n = 1103) is surgery

-----

## Question 2

  - Tokenize the the words in the `transcription` column
  - Count the number of times each token appears
  - Visualize the top 20 most frequent words

Explain what we see from this result. Does it makes sense? What insights
(if any) do we get?

``` r
mt_samples %>%
  unnest_tokens(token, transcription) %>%
  count(token, sort=TRUE) %>%
  top_n(n=20, wt = n) %>%
  ggplot(aes(x=n, y= fct_reorder(token,n)))+
  geom_col()
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

-----

## Question 3

  - Redo visualization but remove stopwords before
  - Bonus points if you remove numbers as well

What do we see know that we have removed stop words? Does it give us a
better idea of what the text is about?

``` r
mt_samples %>%
  unnest_tokens(output = token, input = transcription) %>%
  anti_join(stop_words, by = c("token" = "word")) %>%
  count(token, sort=TRUE) %>%
  top_n(n=20, wt = n) %>%
  ggplot(aes(x=n, y= fct_reorder(token,n)))+
  geom_col()
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

-----

# Question 4

repeat question 2, but this time tokenize into bi-grams. how does the
result change if you look at tri-grams?

``` r
mt_samples %>%
  unnest_ngrams(output=token, input=transcription, n=2) %>%
  count(token, sort=TRUE) %>%
  top_n(n=20, wt = n) %>%
  ggplot(aes(x=n, y= fct_reorder(token,n)))+
  geom_col()
```

## ![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

# Question 5

Using the results you got from question 4. Pick a word and count the
words that appears after and before it.

``` r
mt_samples %>%
  unnest_ngrams(output=token, input=transcription, n=2) %>%
  separate(col=token, into=c("word1","word2"),sep=" ") %>%
  select(word1, word2) %>%
  filter(word1 == "blood") %>%
  count(word2, sort=TRUE)
```

    ## # A tibble: 161 x 2
    ##    word2        n
    ##    <chr>    <int>
    ##  1 pressure  1265
    ##  2 loss       965
    ##  3 cell       130
    ##  4 in         114
    ##  5 cells      112
    ##  6 sugar       91
    ##  7 and         84
    ##  8 sugars      79
    ##  9 was         65
    ## 10 cultures    53
    ## # … with 151 more rows

-----

# Question 6

Which words are most used in each of the specialties. you can use
`group_by()` and `top_n()` from `dplyr` to have the calculations be done
within each specialty. Remember to remove stopwords. How about the most
5 used words?

``` r
mt_samples %>%
  group_by(medical_specialty) %>%
  unnest_tokens(output = token, input = transcription) %>%
  anti_join(stop_words, by = c("token" = "word")) %>%
  count(token, sort=TRUE) %>%
  top_n(n=1, wt = n) 
```

    ## # A tibble: 41 x 3
    ## # Groups:   medical_specialty [40]
    ##    medical_specialty          token       n
    ##    <chr>                      <chr>   <int>
    ##  1 Surgery                    patient  4855
    ##  2 Consult - History and Phy. patient  3046
    ##  3 Orthopedic                 patient  1711
    ##  4 Cardiovascular / Pulmonary left     1550
    ##  5 General Medicine           patient  1356
    ##  6 Gastroenterology           patient   872
    ##  7 Urology                    patient   776
    ##  8 Radiology                  left      701
    ##  9 Emergency Room Reports     patient   685
    ## 10 Discharge Summary          patient   672
    ## # … with 31 more rows

# Question 7 - extra

Find your own insight in the data:

Ideas:

  - Interesting ngrams
  - See if certain words are used more in some specialties then others
