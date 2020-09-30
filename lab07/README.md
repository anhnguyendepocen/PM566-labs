Lab 07 - Web scraping and Regular Expressions
================

``` r
library(stringr)
library(xml2)
library(httr)
knitr::opts_chunk$set(include  = TRUE)
```

# Learning goals

  - Use a real world API to make queries and process the data.
  - Use regular expressions to parse the information.
  - Practice your GitHub skills.

# Lab description

In this lab, we will be working with the [NCBI
API](https://www.ncbi.nlm.nih.gov/home/develop/api/) to make queries and
extract information using XML and regular expressions. For this lab, we
will be using the `httr`, `xml2`, and `stringr` R packages.

This markdown document should be rendered using `github_document`
document.

## Question 1: How many sars-cov-2 papers?

Build an automatic counter of sars-cov-2 papers using PubMed. You will
need to apply XPath as we did during the lecture to extract the number
of results returned by PubMed in the following web address:

    https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2

Complete the lines of code:

``` r
# Downloading the website
website <- xml2::read_html("https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2")
# Finding the counts
counts <- xml2::xml_find_first(website, "/html/body/main/div[9]/div[2]/div[2]/div[1]/span")
# Turning it into text
counts <- as.character(counts)
# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]+")
```

    ## [1] "33,814"

## Question 2: Academic publications on COVID19 and Hawaii

You need to query the following The parameters passed to the query are
documented [here](https://www.ncbi.nlm.nih.gov/books/NBK25499/).

Use the function `httr::GET()` to make the following query:

1.  Baseline URL:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi>

2.  Query parameters:
    
      - db: pubmed
      - term: covid19 hawaii
      - retmax: 1000

<!-- end list -->

``` r
library(httr)
query_ids <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi",
  query = list(db="pubmed",term="covid19 hawaii", retmax=1000)
)
# Extracting the content of the response of GET
ids <- httr::content(query_ids)
```

The query will return an XML object, we can turn it into a character
list to analyze the text directly with `as.character()`. Another way of
processing the data could be using lists with the function
`xml2::as_list()`. We will skip the latter for now.

Take a look at the data, and continue with the next question (don’t
forget to commit and push your results to your GitHub repo\!).

## Question 3: Get details about the articles

The Ids are wrapped around text in the following way: `<Id>... id number
...</Id>`. we can use a regular expression that extract that
information. Fill out the following lines of code:

``` r
# Turn the result into a character vector
ids <- as.character(ids)
# Find all the ids 
ids <- stringr::str_extract_all(ids, "<Id>[0-9]+</Id>")[[1]]
# Remove all the leading and trailing <Id> </Id>. Make use of "|"
ids <- stringr::str_remove_all(ids, "<Id>|</Id>")
 paste(ids,collapse=",")
```

    ## [1] "32984015,32969950,32921878,32914097,32914093,32912595,32907823,32907673,32888905,32881116,32837709,32763956,32763350,32745072,32742897,32692706,32690354,32680824,32666058,32649272,32596689,32592394,32584245,32501143,32486844,32462545,32432219,32432218,32432217,32427288,32420720,32386898,32371624,32371551,32361738,32326959,32323016,32314954,32300051,32259247"

With the ids in hand, we can now try to get the abstracts of the papers.
As before, we will need to coerce the contents (results) to a list
using:

1.  Baseline url:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi>

2.  Query parameters:
    
      - db: pubmed
      - id: A character with all the ids separated by comma, e.g.,
        “1232131,546464,13131”
      - retmax: 1000
      - rettype: abstract

**Pro-tip**: If you want `GET()` to take some element literal, wrap it
around `I()` (as you would do in a formula in R). For example, the text
`"123,456"` is replaced with `"123%2C456"`. If you don’t want that
behavior, you would need to do the following `I("123,456")`.

``` r
publications <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi",
  query = list(db="pubmed", 
               id=paste(ids,collapse=","),
               retmax=1000, 
               rettype="abstract")
)
# Turning the output into character vector
publications <- httr::content(publications)
publications_txt <- as.character(publications)
```

With this in hand, we can now analyze the data. This is also a good time
for committing and pushing your work\!

## Question 4: Distribution of universities, schools, and departments

Using the function `stringr::str_extract_all()` applied on
`publications_txt`, capture all the terms of the form:

1.  University of …
2.  … Institute of …

Write a regular expression that captures all such instances

``` r
institution <- str_extract_all(
  publications_txt,
  "University of [[:alpha:]]+|[[:alpha:]]+ Institute of [[:alpha:]]+"
  ) 
institution <- unlist(institution)
table(institution)
```

    ## institution
    ##      Australian Institute of Tropical Massachusetts Institute of Technology 
    ##                                     9                                     1 
    ##   National Institute of Environmental    Prophylactic Institute of Southern 
    ##                                     3                                     2 
    ##                 University of Arizona              University of California 
    ##                                     2                                     6 
    ##                 University of Chicago                University of Colorado 
    ##                                     1                                     1 
    ##                   University of Hawai                  University of Hawaii 
    ##                                    20                                    38 
    ##                  University of Health                University of Illinois 
    ##                                     1                                     1 
    ##                    University of Iowa                University of Lausanne 
    ##                                     4                                     1 
    ##              University of Louisville                University of Nebraska 
    ##                                     1                                     5 
    ##                  University of Nevada                     University of New 
    ##                                     1                                     2 
    ##            University of Pennsylvania              University of Pittsburgh 
    ##                                    18                                     5 
    ##                 University of Science                   University of South 
    ##                                    14                                     1 
    ##                University of Southern                  University of Sydney 
    ##                                     1                                     1 
    ##                   University of Texas                     University of the 
    ##                                     5                                     1 
    ##                    University of Utah               University of Wisconsin 
    ##                                     2                                     3

Repeat the exercise and this time focus on schools and departments in
the form of

1.  School of …
2.  Department of …

And tabulate the results

``` r
schools_and_deps <- str_extract_all(
  publications_txt,
   "School of [[:alpha:]]+|[[:alpha:]]+ Department of [[:alpha:]]+"
  )
table(schools_and_deps)
```

    ## schools_and_deps
    ## Abramson Department of Rehabilitation     Faillace Department of Psychiatry 
    ##                                     1                                     1 
    ##        Hospital Department of Surgery                    School of Medicine 
    ##                                     1                                    87 
    ##                     School of Natural                     School of Nursing 
    ##                                     1                                     1 
    ##                      School of Public                      School of Social 
    ##                                    20                                     1 
    ##         States Department of Veterans       the Department of Communication 
    ##                                     1                                     1 
    ##      University Department of Surgery             US Department of Veterans 
    ##                                     2                                     1

## Question 5: Form a database

We want to build a dataset which includes the title and the abstract of
the paper. The title of all records is enclosed by the HTML tag
`ArticleTitle`, and the abstract by `Abstract`.

Before applying the functions to extract text directly, it will help to
process the XML a bit. We will use the `xml2::xml_children()` function
to keep one element per id. This way, if a paper is missing the
abstract, or something else, we will be able to properly match PUBMED
IDS with their corresponding records.

``` r
pub_char_list <- xml2::xml_children(publications)
pub_char_list <- sapply(pub_char_list, as.character)
```

Now, extract the abstract and article title for each one of the elements
of `pub_char_list`. You can either use `sapply()` as we just did, or
simply take advantage of vectorization of `stringr::str_extract`

``` r
abstracts <- str_extract(pub_char_list, "[YOUR REGULAR EXPRESSION]")
abstracts <- str_remove_all(abstracts, "[CLEAN ALL THE HTML TAGS]")
abstracts <- str_remove_all(abstracts, "[CLEAN ALL EXTRA WHITE SPACE AND NEW LINES]")
```

How many of these don’t have an abstract? Now, the title

``` r
titles <- str_extract(pub_char_list, "[YOUR REGULAR EXPRESSION]")
titles <- str_remove_all(titles, "[CLEAN ALL THE HTML TAGS]")
```

Finally, put everything together into a single `data.frame` and use
`knitr::kable` to print the results

``` r
database <- data.frame(
  "[DATA TO CONCATENATE]"
)
knitr::kable(database)
```

Done\! Knit the document, commit, and push.

## Final Pro Tip (optional)

You can still share the HTML document on github. You can include a link
in your `README.md` file as the following:

``` md
View [here](https://ghcdn.rawgit.org/:user/:repo/:tag/:file)
```

For example, if we wanted to add a direct link the HTML page of lecture
7, we could do something like the
following:

``` md
View [here](https://ghcdn.rawgit.org/USCbiostats/PM566/master/static/slides/07-apis-regex/slides.html)
```
