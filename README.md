Lab 05 - Data Wrangling
================

# Learning goals

  - Use the `merge()` function to join two datasets.
  - Deal with missings and impute data.
  - Identify relevant observations using `quantile()`.
  - Practice your GitHub skills.

# Lab description

For this lab we will be, again, dealing with the meteorological dataset
downloaded from the NOAA, the `met`. In this case, we will use
`data.table` to answer some questions regarding the `met` dataset, while
at the same time practice your Git+GitHub skills for this project.

This markdown document should be rendered using `github_document`
document.

# Part 1: Setup the Git project and the GitHub repository

1.  Go to your documents (or wherever you are planning to store the
    data) in your computer, and create a folder for this project, for
    example, “PM566-labs”

2.  In that folder, save [this
    template](https://raw.githubusercontent.com/USCbiostats/PM566/master/content/assignment/05-lab.Rmd)
    as “README.Rmd”. This will be the markdown file where all the magic
    will happen.

3.  Go to your GitHub account and create a new repository, hopefully of
    the same name that this folder has, i.e., “PM566-labs”.

4.  Initialize the Git project, add the “README.Rmd” file, and make your
    first commit.

5.  Add the repo you just created on GitHub.com to the list of remotes,
    and push your commit to origin while setting the upstream.

Most of the steps can be done using command line:

``` sh
# Step 1
cd ~/Documents
mkdir PM566-labs
cd PM566-labs

# Step 2
wget https://raw.githubusercontent.com/USCbiostats/PM566/master/content/assignment/05-lab.Rmd 
mv 05-lab.Rmd README.md

# Step 3
# Happens on github

# Step 4
git init
git add README.Rmd
git commit -m "First commit"

# Step 5
git remote add origin git@github.com:[username]/PM566-labs
git push -u origin master
```

You can also complete the steps in R (replace with your paths/username
when needed)

``` r
# Step 1
setwd("~/Documents")
dir.create("PM566-labs")
setwd("PM566-labs")

# Step 2
download.file(
  "https://raw.githubusercontent.com/USCbiostats/PM566/master/content/assignment/05-lab.Rmd",
  destfile = "README.Rmd"
  )

# Step 3: Happens on Github

# Step 4
system("git init && git add README.Rmd")
system('git commit -m "First commit"')

# Step 5
system("git remote add origin git@github.com:[username]/PM566-labs")
system("git push -u origin master")
```

Once you are done setting up the project, you can now start working with
the MET data.

## Setup in R

1.  Load the `data.table` (and the `dtplyr` and `dplyr` packages if you
    plan to work with those).

<!-- end list -->

``` r
library(data.table)
met<-fread("/Users/meredith/Dropbox (University of Southern California)/Courses/PM566/met_all.gz")
```

2.  Load the met data from
    <https://raw.githubusercontent.com/USCbiostats/data-science-data/master/02_met/met_all.gz>,
    and also the station data. For the later, you can use the code we
    used during lecture to pre-process the stations data:

<!-- end list -->

``` r
# Download the data
stations <- fread("ftp://ftp.ncdc.noaa.gov/pub/data/noaa/isd-history.csv")
stations[, USAF := as.integer(USAF)]
```

    ## Warning in eval(jsub, SDenv, parent.frame()): NAs introduced by coercion

``` r
# Dealing with NAs and 999999
stations[, USAF   := fifelse(USAF == 999999, NA_integer_, USAF)]
stations[, CTRY   := fifelse(CTRY == "", NA_character_, CTRY)]
stations[, STATE  := fifelse(STATE == "", NA_character_, STATE)]

# Selecting the three relevant columns, and keeping unique records
stations <- unique(stations[, list(USAF, CTRY, STATE)])

# Dropping NAs
stations <- stations[!is.na(USAF)]

# Removing duplicates
stations[, n := 1:.N, by = .(USAF)]
stations <- stations[n == 1,][, n := NULL]
```

3.  Merge the data as we did during the lecture.

## Question 1: Representative station for the US

What is the median station in terms of temperature, wind speed, and
atmospheric pressure? Look for the three weather stations that best
represent continental US using the `quantile()` function. Do these three
coincide?

``` r
met<-merge(x=met, y=stations, by.x="USAFID", by.y = "USAF", all.x=TRUE, all.y=FALSE)
names(met)
```

    ##  [1] "USAFID"            "WBAN"              "year"             
    ##  [4] "month"             "day"               "hour"             
    ##  [7] "min"               "lat"               "lon"              
    ## [10] "elev"              "wind.dir"          "wind.dir.qc"      
    ## [13] "wind.type.code"    "wind.sp"           "wind.sp.qc"       
    ## [16] "ceiling.ht"        "ceiling.ht.qc"     "ceiling.ht.method"
    ## [19] "sky.cond"          "vis.dist"          "vis.dist.qc"      
    ## [22] "vis.var"           "vis.var.qc"        "temp"             
    ## [25] "temp.qc"           "dew.point"         "dew.point.qc"     
    ## [28] "atm.press"         "atm.press.qc"      "rh"               
    ## [31] "CTRY"              "STATE"

``` r
table(met$STATE)
```

    ## 
    ##     AL     AR     AZ     CA     CO     CT     DE     FL     GA     IA     ID 
    ##  44743  34829  34150 109392  78843  11767   3231  80933  89241 107995  16005 
    ##     IL     IN     KS     KY     LA     MA     MD     ME     MI     MN     MO 
    ##  73792  38489  40707  30482  60958  20823  21323  12850 119490 120067  36229 
    ##     MS     MT     NC     ND     NE     NH     NJ     NM     NV     NY     OH 
    ##  33598   7654 100807  39207  65209  10862  16954  37106  18914  34164  51853 
    ##     OK     OR     PA     RI     SC     SD     TN     TX     UT     VA     VT 
    ##  80623  10484  47541   4879  63770  26339  21950 248410  20957  83500  14883 
    ##     WA     WI     WV     WY 
    ##   6731  97544  15396  31669

Knit the document, commit your changes, and Save it on GitHub. Don’t
forget to add `README.md` to the tree, the first time you render it.

## Question 2: Representative station per state

Just like the previous question, you are asked to identify what is the
most representative, the median, station per state. This time, instead
of looking at one variable at a time, look at the euclidean distance. If
multiple stations show in the median, select the one located at the
lowest latitude.

Knit the doc and save it on GitHub.

## Question 3: In the middle?

For each state, identify what is the station that is closest to the
mid-point of the state. Combining these with the stations you identified
in the previous question, use `leaflet()` to visualize all ~100 points
in the same figure, applying different colors for those identified in
this question.

Knit the doc and save it on GitHub.

## Question 4: Means of means

Using the `quantile()` function, generate a summary table that shows the
number of states included, average temperature, wind-speed, and
atmospheric pressure by the variable “average temperature level,” which
you’ll need to create.

Start by computing the states’ average temperature. Use that measurement
to classify them according to the following criteria:

  - low: temp \< 20
  - Mid: temp \>= 20 and temp \< 25
  - High: temp \>= 25

Once you are done with that, you can compute the following:

  - Number of entries (records),
  - Number of NA entries,
  - Number of stations,
  - Number of states included, and
  - Mean temperature, wind-speed, and atmospheric pressure.

All by the levels described before.

Knit the document, commit your changes, and push them to GitHub. If
you’d like, you can take this time to include the link of [the issue
of the week](https://github.com/USCbiostats/PM566/issues/23) so that you
let us know when you are done,
e.g.,

``` bash
git commit -a -m "Finalizing lab 5 https://github.com/USCbiostats/PM566/issues/23"
```
