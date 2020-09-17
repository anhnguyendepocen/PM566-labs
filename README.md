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

3.  Merge the data as we did during the
lecture.

<!-- end list -->

``` r
met<-merge(x=met, y=stations, by.x="USAFID", by.y = "USAF", all.x=TRUE, all.y=FALSE)
```

## Question 1: Representative station for the US

What is the median station in terms of temperature, wind speed, and
atmospheric pressure? Look for the three weather stations that best
represent continental US using the `quantile()` function. Do these three
coincide?

``` r
# obtain station averages
met_stations <- met[, .(
  wind.sp = mean(wind.sp,na.rm=TRUE),
  temp=mean(temp, na.rm=TRUE),
  atm.press= mean(atm.press,na.rm=TRUE)
  ), by=.(USAFID, STATE)]

# obtain median
met_stations[,temp50 := quantile(temp, probs=0.5, na.rm=TRUE)]
met_stations[,atm.press50 := quantile(atm.press, probs=0.5, na.rm=TRUE)]
met_stations[,wind.sp50 := quantile(wind.sp, probs=0.5, na.rm=TRUE)]

# filter the data
met_stations[which.min(abs(temp-temp50))]
```

    ##    USAFID STATE  wind.sp     temp atm.press   temp50 atm.press50 wind.sp50
    ## 1: 720458    KY 1.209682 23.68173       NaN 23.68406    1014.691  2.461838

``` r
met_stations[which.min(abs(atm.press-atm.press50))]
```

    ##    USAFID STATE  wind.sp     temp atm.press   temp50 atm.press50 wind.sp50
    ## 1: 722238    AL 1.472656 26.13978  1014.691 23.68406    1014.691  2.461838

``` r
met_stations[which.min(abs(wind.sp-wind.sp50))]
```

    ##    USAFID STATE  wind.sp     temp atm.press   temp50 atm.press50 wind.sp50
    ## 1: 720929    WI 2.461838 17.43278       NaN 23.68406    1014.691  2.461838

No, the three stations do not coincide.

Knit the document, commit your changes, and Save it on GitHub. Don’t
forget to add `README.md` to the tree, the first time you render it.

## Question 2: Representative station per state

Just like the previous question, you are asked to identify what is the
most representative, the median, station per state. This time, instead
of looking at one variable at a time, look at the euclidean distance. If
multiple stations show in the median, select the one located at the
lowest latitude.

``` r
# obtain median by sate
met_stations[,temp50s := quantile(temp, probs=0.5, na.rm=TRUE), by=STATE]
met_stations[,atm.press50s := quantile(atm.press, probs=0.5, na.rm=TRUE), by=STATE]
met_stations[,wind.sp50s := quantile(wind.sp, probs=0.5, na.rm=TRUE), by=STATE]

met_stations[, tempdif := which.min(abs(temp-temp50s)), by=STATE]
met_stations[, recordid :=1:.N, by=STATE]
met_stations[recordid==tempdif,.(USAFID, temp, temp50s, STATE)]
```

    ##     USAFID     temp  temp50s STATE
    ##  1: 720202 17.16329 17.98061    OR
    ##  2: 720254 19.24684 19.24684    WA
    ##  3: 720284 20.51970 20.51970    MI
    ##  4: 720328 21.94820 21.94446    WV
    ##  5: 720545 22.44858 22.36880    CT
    ##  6: 720592 26.31534 26.33664    AL
    ##  7: 720605 25.87364 25.80545    SC
    ##  8: 720636 23.99322 23.95109    MO
    ##  9: 720855 18.45570 18.52849    ND
    ## 10: 720964 27.57697 27.57325    FL
    ## 11: 722041 27.84758 27.87430    LA
    ## 12: 722133 27.14427 27.14427    OK
    ## 13: 722142 20.32324 20.56798    ID
    ## 14: 722188 26.07275 26.24296    AR
    ## 15: 722197 26.70404 26.70404    GA
    ## 16: 722218 24.89883 24.89883    MD
    ## 17: 722322 23.98226 23.88844    KY
    ## 18: 722358 26.54093 26.69258    MS
    ## 19: 722550 29.74982 29.75188    TX
    ## 20: 722692 24.37799 24.37799    VA
    ## 21: 722745 30.31538 30.32372    AZ
    ## 22: 722931 22.66268 22.66268    CA
    ## 23: 723060 24.70791 24.72953    NC
    ## 24: 723273 25.01262 24.88657    TN
    ## 25: 723658 24.94447 24.94447    NM
    ## 26: 724090 23.47238 23.47238    NJ
    ## 27: 724180 24.56026 24.56026    DE
    ## 28: 724200 22.03309 22.02062    OH
    ## 29: 724386 22.32575 22.25059    IN
    ## 30: 724555 24.21648 24.21220    KS
    ## 31: 724699 21.94228 21.49638    CO
    ## 32: 724855 24.34157 24.56293    NV
    ## 33: 724988 20.44142 20.40674    NY
    ## 34: 725064 21.40933 21.30662    MA
    ## 35: 725070 22.53551 22.53551    RI
    ## 36: 725130 21.69177 21.69177    PA
    ## 37: 725305 22.36831 22.43194    IL
    ## 38: 725526 21.87354 21.87354    NE
    ## 39: 725570 21.36209 21.33461    IA
    ## 40: 725724 24.39332 24.35182    UT
    ## 41: 726073 18.82098 18.79016    ME
    ## 42: 726115 18.60548 18.61379    VT
    ## 43: 726116 19.23920 19.55054    NH
    ## 44: 726438 18.85524 18.85524    WI
    ## 45: 726589 19.58483 19.63017    MN
    ## 46: 726627 20.35662 20.35662    SD
    ## 47: 726650 19.75554 19.80699    WY
    ## 48: 726777 19.15492 19.15492    MT
    ##     USAFID     temp  temp50s STATE

``` r
met_stations[, atmdif := which.min(abs(atm.press-atm.press50s)), by=STATE]
met_stations[recordid==atmdif,.(USAFID, temp, temp50s, STATE)]
```

    ##     USAFID     temp  temp50s STATE
    ##  1: 722029 28.08069 27.57325    FL
    ##  2: 722085 27.72732 25.80545    SC
    ##  3: 722093 16.89136 20.51970    MI
    ##  4: 722181 27.03306 26.70404    GA
    ##  5: 722269 26.68915 26.33664    AL
    ##  6: 722320 27.90102 27.87430    LA
    ##  7: 722340 26.91166 26.69258    MS
    ##  8: 722479 30.33387 29.75188    TX
    ##  9: 722745 30.31538 30.32372    AZ
    ## 10: 722899 24.41995 22.66268    CA
    ## 11: 723109 25.91019 24.72953    NC
    ## 12: 723300 25.05247 23.95109    MO
    ## 13: 723346 24.59407 24.88657    TN
    ## 14: 723436 24.42281 26.24296    AR
    ## 15: 723537 27.05520 27.14427    OK
    ## 16: 723600 23.31571 24.94447    NM
    ## 17: 724037 24.62280 24.37799    VA
    ## 18: 724040 25.58791 24.89883    MD
    ## 19: 724075 23.83986 23.47238    NJ
    ## 20: 724120 20.81074 21.94446    WV
    ## 21: 724180 24.56026 24.56026    DE
    ## 22: 724237 24.55104 23.88844    KY
    ## 23: 724286 22.04699 22.02062    OH
    ## 24: 724373 23.13695 22.25059    IN
    ## 25: 724586 24.46104 24.21220    KS
    ## 26: 724660 22.56250 21.49638    CO
    ## 27: 724860 20.95717 24.56293    NV
    ## 28: 725040 23.43920 22.36880    CT
    ## 29: 725053 23.52960 20.40674    NY
    ## 30: 725064 21.40933 21.30662    MA
    ## 31: 725070 22.53551 22.53551    RI
    ## 32: 725109 22.28661 21.69177    PA
    ## 33: 725440 22.84806 22.43194    IL
    ## 34: 725461 20.32871 21.33461    IA
    ## 35: 725555 21.10637 21.87354    NE
    ## 36: 725686 20.91280 19.80699    WY
    ## 37: 725755 24.31031 24.35182    UT
    ## 38: 725784 20.87058 20.56798    ID
    ## 39: 725895 18.79793 17.98061    OR
    ## 40: 726114 17.46999 18.61379    VT
    ## 41: 726155 19.96899 19.55054    NH
    ## 42: 726196 18.75935 18.79016    ME
    ## 43: 726425 19.45558 18.85524    WI
    ## 44: 726545 21.36855 20.35662    SD
    ## 45: 726559 20.04596 19.63017    MN
    ## 46: 726777 19.15492 19.15492    MT
    ##     USAFID     temp  temp50s STATE

``` r
met_stations[, wsdif:=which.min(abs(wind.sp-wind.sp50s)), by=STATE]
met_stations[recordid==wsdif,.(USAFID, temp, temp50s, STATE)]
```

    ##     USAFID     temp  temp50s STATE
    ##  1: 720254 19.24684 19.24684    WA
    ##  2: 720328 21.94820 21.94446    WV
    ##  3: 720386 20.07851 19.63017    MN
    ##  4: 720422 22.25238 24.21220    KS
    ##  5: 720492 17.87100 18.61379    VT
    ##  6: 720532 20.57839 21.49638    CO
    ##  7: 720602 26.80807 25.80545    SC
    ##  8: 720858 17.93556 18.52849    ND
    ##  9: 720951 27.06469 26.70404    GA
    ## 10: 720971 17.46586 19.80699    WY
    ## 11: 721031 24.36992 24.88657    TN
    ## 12: 722029 28.08069 27.57325    FL
    ## 13: 722076 22.34403 22.43194    IL
    ## 14: 722165 24.54241 26.69258    MS
    ## 15: 722202 30.19653 29.75188    TX
    ## 16: 722218 24.89883 24.89883    MD
    ## 17: 722275 27.83985 26.33664    AL
    ## 18: 722486 28.16413 27.87430    LA
    ## 19: 722676 29.61129 24.94447    NM
    ## 20: 722740 31.42383 30.32372    AZ
    ## 21: 722899 24.41995 22.66268    CA
    ## 22: 723010 23.42556 24.72953    NC
    ## 23: 723415 27.84015 26.24296    AR
    ## 24: 723545 27.03555 27.14427    OK
    ## 25: 723860 34.78496 24.56293    NV
    ## 26: 724006 24.31662 24.37799    VA
    ## 27: 724090 23.47238 23.47238    NJ
    ## 28: 724180 24.56026 24.56026    DE
    ## 29: 724303 22.75657 22.02062    OH
    ## 30: 724350 25.02776 23.88844    KY
    ## 31: 724373 23.13695 22.25059    IN
    ## 32: 724458 24.75847 23.95109    MO
    ## 33: 724700 24.16964 24.35182    UT
    ## 34: 725016 22.67545 20.40674    NY
    ## 35: 725079 22.27697 22.53551    RI
    ## 36: 725087 22.57539 22.36880    CT
    ## 37: 725088 21.20391 21.30662    MA
    ## 38: 725103 23.22796 21.69177    PA
    ## 39: 725464 21.37948 21.33461    IA
    ## 40: 725624 21.68166 21.87354    NE
    ## 41: 725867 20.81272 20.56798    ID
    ## 42: 725975 16.97502 17.98061    OR
    ## 43: 726056 20.52602 19.55054    NH
    ## 44: 726077 18.49969 18.79016    ME
    ## 45: 726284 15.33980 20.51970    MI
    ## 46: 726504 16.32511 18.85524    WI
    ## 47: 726519 19.03976 20.35662    SD
    ## 48: 726770 22.99419 19.15492    MT
    ##     USAFID     temp  temp50s STATE

Knit the doc and save it on GitHub.

## Question 3: In the middle?

For each state, identify what is the station that is closest to the
mid-point of the state. Combining these with the stations you identified
in the previous question, use `leaflet()` to visualize all ~100 points
in the same figure, applying different colors for those identified in
this question.

``` r
met_stations <- unique(met[,.(USAFID, STATE, lat, lon)])
met_stations[,n :=1:.N, by=USAFID]
met_stations <-met_stations[n==1]

met_stations <- met[, lat_mid := quantile(lat, probs=.5, na.rm=TRUE),by=STATE]
met_stations <- met[, lon_mid := quantile(lon, probs=.5, na.rm=TRUE),by=STATE]

# look at distances
met_stations[,distance := sqrt((lat-lat_mid)^2 + (lon-lon_mid)^2)]
met_stations[,minrecord := which.min(distance), by=STATE]
met_stations[,n :=1:.N,by=STATE]
met_stations[n==minrecord]
```

    ##     USAFID  WBAN year month day hour min    lat      lon elev wind.dir
    ##  1: 720328 63832 2019     8   1    0  15 39.000  -80.274  498      100
    ##  2: 720388   469 2019     8   1    0  15 47.104 -122.287  164       NA
    ##  3: 720448   144 2019     8   1    0  15 37.578  -84.770  312       NA
    ##  4: 720468   466 2019     8   1    0  15 30.558  -92.099   23      160
    ##  5: 720498   153 2019     8   1    0  56 37.400  -77.517   72       90
    ##  6: 720545   169 2019     8   1    0  15 41.384  -72.506  127       NA
    ##  7: 720603   195 2019     8   1    0  15 34.283  -80.567   92       NA
    ##  8: 720867   293 2019     8   1    0  15 48.390 -100.024  472      190
    ##  9: 720928   315 2019     8   1    0  15 40.280  -83.115  288       10
    ## 10: 720961   336 2019     8   1    0  15 40.711  -86.375  225      340
    ## 11: 721031   348 2019     8   1    0  15 35.380  -86.246  330      180
    ## 12: 721042   486 2019     8   1    0  15 28.228  -82.156   27      140
    ## 13: 722061  3038 2019     8   1    0  14 39.467 -106.150 3680      100
    ## 14: 722201  3723 2019     8   9   11  35 35.584  -79.101   75       NA
    ## 15: 722217 63881 2019     8   1    0  15 32.564  -82.985   94       NA
    ## 16: 722300 53864 2019     8  13    0  53 33.177  -86.783  179      220
    ## 17: 722570  3933 2019     8   1    0  58 31.150  -97.717  282       90
    ## 18: 722683 93083 2019     8   1    8  55 33.463 -105.535 2077       NA
    ## 19: 723429 53920 2019     8  13    0  53 35.259  -93.093  123       NA
    ## 20: 723540 13919 2019     8   1    0  56 35.417  -97.383  393      160
    ## 21: 723745   374 2019     8   1    0  15 34.257 -111.339 1572       NA
    ## 22: 724060 93721 2019     8   1    0  54 39.173  -76.684   47       NA
    ## 23: 724088 13707 2019     8   1    0  18 39.133  -75.467    9       70
    ## 24: 724090 14780 2019     8   5   22   0 40.033  -74.353   31       90
    ## 25: 724397 54831 2019     8   9   11  56 40.477  -88.916  265      350
    ## 26: 724453  3994 2019     8   1    0  53 38.704  -93.183  276      350
    ## 27: 724509 53939 2019     8   1    6  56 38.068  -97.275  467      120
    ## 28: 724750 23176 2019     8   1    0  52 38.427 -113.012 1536       NA
    ## 29: 724770  3170 2019     8   1   12  53 39.600 -116.010 1812      200
    ## 30: 724815 23257 2019     8   1    0  53 37.285 -120.512   48      310
    ## 31: 725023   474 2019     8   9   11  35 33.761  -90.758   43       NA
    ## 32: 725068 54777 2019     8   1    0  52 41.876  -71.021   13       NA
    ## 33: 725074 54752 2019     8   1    0  50 41.597  -71.412    5       NA
    ## 34: 725118 14751 2019     8   1    0  56 40.217  -76.851  106      240
    ## 35: 725150  4725 2019     8  13    1  53 42.209  -75.980  499      210
    ## 36: 725405 54816 2019     8   1    0  15 43.322  -84.688  230       90
    ## 37: 725466  4938 2019     8   1    0  15 41.691  -93.566  277       NA
    ## 38: 725513  4901 2019     8   1    0  15 40.893  -97.997  550      110
    ## 39: 725975 24235 2019     8   1    0  56 42.600 -123.364 1171       NA
    ## 40: 726050 14745 2019     8   1    0  51 43.205  -71.503  105      160
    ## 41: 726114 54771 2019     8   1   18  54 44.535  -72.614  223       NA
    ## 42: 726185 14605 2019     8   1    0  53 44.316  -69.797  110      290
    ## 43: 726465 94890 2019     8   1    0  55 44.778  -89.667  389       NA
    ## 44: 726530 94943 2019     8   1    0  15 43.767  -99.318  517      130
    ## 45: 726583  4941 2019     8   1    0  55 45.097  -94.507  347      120
    ## 46: 726720 24061 2019     8   1    0  53 43.064 -108.458 1684      150
    ## 47: 726798 24150 2019     8  13    0  53 45.699 -110.448 1420      280
    ## 48: 726810 24131 2019     8   1    0  53 43.567 -116.240  874      330
    ##     USAFID  WBAN year month day hour min    lat      lon elev wind.dir
    ##     wind.dir.qc wind.type.code wind.sp wind.sp.qc ceiling.ht ceiling.ht.qc
    ##  1:           5              N     1.5          5         NA             9
    ##  2:           9              C     0.0          5      22000             5
    ##  3:           9              C     0.0          5      22000             5
    ##  4:           5              N     2.6          5      22000             5
    ##  5:           1              N     1.5          1      22000             5
    ##  6:           9              C     0.0          5      22000             5
    ##  7:           9              C     0.0          5      22000             5
    ##  8:           5              N     3.6          5      22000             5
    ##  9:           5              N     3.1          5      22000             5
    ## 10:           5              N     3.6          5      22000             5
    ## 11:           5              N     2.6          5       1524             5
    ## 12:           5              N     3.6          5       3353             5
    ## 13:           5              N     5.7          5       2896             5
    ## 14:           9              C     0.0          1      22000             1
    ## 15:           9              C     0.0          5      22000             5
    ## 16:           1              N     1.5          1       2134             1
    ## 17:           1              N     4.1          1      22000             1
    ## 18:           9              C     0.0          1         NA             9
    ## 19:           9              C     0.0          1      22000             1
    ## 20:           5              N     3.6          5      22000             5
    ## 21:           9              C     0.0          5       3658             5
    ## 22:           9              C     0.0          5       4877             5
    ## 23:           5              N     5.1          5       2134             5
    ## 24:           1              N     3.6          1      22000             1
    ## 25:           1              N     2.6          1      22000             1
    ## 26:           5              N     5.1          5       3048             5
    ## 27:           5              N     6.2          5         NA             9
    ## 28:           9              C     0.0          1      22000             1
    ## 29:           1              N     1.5          1         NA             9
    ## 30:           5              N     4.1          5      22000             5
    ## 31:           9              C     0.0          1      22000             5
    ## 32:           9              C     0.0          5      22000             5
    ## 33:           9              C     0.0          5       4572             5
    ## 34:           5              N     3.1          5      22000             5
    ## 35:           1              N     3.1          1      22000             5
    ## 36:           5              N     1.5          5      22000             5
    ## 37:           9              N      NA          9      22000             5
    ## 38:           5              N     5.7          5      22000             5
    ## 39:           9              V     1.5          5      22000             5
    ## 40:           5              N     1.5          5      22000             5
    ## 41:           9              V     3.1          1      22000             1
    ## 42:           5              N     2.1          5       1676             5
    ## 43:           9              C     0.0          1      22000             1
    ## 44:           5              N     5.1          5      22000             5
    ## 45:           1              N     3.1          1      22000             5
    ## 46:           5              N     3.1          5      22000             5
    ## 47:           1              N     6.2          1      22000             1
    ## 48:           5              N     4.1          5      22000             5
    ##     wind.dir.qc wind.type.code wind.sp wind.sp.qc ceiling.ht ceiling.ht.qc
    ##     ceiling.ht.method sky.cond vis.dist vis.dist.qc vis.var vis.var.qc temp
    ##  1:                 9        N    16093           5       N          5 22.0
    ##  2:                 9        N    16093           5       N          5 29.0
    ##  3:                 9        N    16093           5       N          5 26.0
    ##  4:                 9        N    16093           5       N          5 27.8
    ##  5:                 9        N    16093           1       9          9 23.3
    ##  6:                 9        N    11265           5       N          5 21.0
    ##  7:                 9        N    16093           5       N          5 29.0
    ##  8:                 9        N    16093           5       N          5 29.0
    ##  9:                 9        N    16093           5       N          5 25.0
    ## 10:                 9        N    16093           5       N          5 22.0
    ## 11:                 M        N    16093           5       N          5 26.0
    ## 12:                 M        N    11265           5       N          5 22.7
    ## 13:                 M        N    16093           5       N          5 15.0
    ## 14:                 9        N    16093           1       9          9 20.0
    ## 15:                 9        N    16093           5       N          5 29.0
    ## 16:                 9        N    16093           1       9          9 32.2
    ## 17:                 9        N    16093           1       9          9 34.5
    ## 18:                 9        N    16093           1       9          9 20.0
    ## 19:                 9        N    16093           1       9          9 30.0
    ## 20:                 9        N    16093           5       N          5 32.3
    ## 21:                 M        N    16093           5       N          5 26.0
    ## 22:                 M        N    16093           5       N          5 26.7
    ## 23:                 M        N    16093           5       N          5 26.0
    ## 24:                 9        N       NA           9       9          9 26.1
    ## 25:                 9        N    16093           1       9          9 17.8
    ## 26:                 M        N    16093           5       N          5 21.1
    ## 27:                 9        N    16093           5       N          5 25.0
    ## 28:                 9        N    16093           1       9          9 24.4
    ## 29:                 9        N       NA           9       9          9 12.2
    ## 30:                 9        N    16093           5       N          5 35.0
    ## 31:                 9        N    16093           1       9          9 24.8
    ## 32:                 9        N    16093           5       N          5 23.3
    ## 33:                 M        N    16093           5       N          5 25.0
    ## 34:                 9        N    16093           5       N          5 26.7
    ## 35:                 9        N    16093           1       9          9 21.7
    ## 36:                 9        N    16093           5       N          5 22.7
    ## 37:                 9        N    16093           5       N          5 19.0
    ## 38:                 9        N    16093           5       N          5 26.6
    ## 39:                 9        N    16093           5       N          5 25.0
    ## 40:                 9        N     8047           5       N          5 21.1
    ## 41:                 9        N    16093           1       9          9 24.4
    ## 42:                 M        N     8047           5       N          5 22.2
    ## 43:                 9        N    16093           1       9          9 21.0
    ## 44:                 9        N    16093           5       N          5 26.2
    ## 45:                 9        N    16093           1       9          9 25.0
    ## 46:                 9        N    16093           5       N          5 27.2
    ## 47:                 9        N    16093           1       9          9 23.3
    ## 48:                 9        N    16093           5       N          5 36.1
    ##     ceiling.ht.method sky.cond vis.dist vis.dist.qc vis.var vis.var.qc temp
    ##     temp.qc dew.point dew.point.qc atm.press atm.press.qc        rh CTRY STATE
    ##  1:       5      18.0            5        NA            9  78.14902   US    WV
    ##  2:       5      10.0            5        NA            9  30.60164   US    WA
    ##  3:       5      20.0            5        NA            9  69.58648   US    KY
    ##  4:       5      23.2            5        NA            9  76.09515   US    LA
    ##  5:       1      17.2            1    1018.4            1  68.68188   US    VA
    ##  6:       5      21.0            5        NA            9 100.00000   US    CT
    ##  7:       5      22.0            5        NA            9  65.95870   US    SC
    ##  8:       5      21.0            5        NA            9  62.03959   US    ND
    ##  9:       5      18.0            5        NA            9  65.20835   US    OH
    ## 10:       5      15.0            5        NA            9  64.62394   US    IN
    ## 11:       5      22.0            5        NA            9  78.67081   US    TN
    ## 12:       5      22.1            5        NA            9  96.43240   US    FL
    ## 13:       5       5.0            5        NA            9  51.46430   US    CO
    ## 14:       1      19.9            1        NA            9  99.38627   US    NC
    ## 15:       5      20.0            5        NA            9  58.32577   US    GA
    ## 16:       1      22.8            1    1011.5            1  57.57832   US    AL
    ## 17:       1      18.6            1        NA            9  38.96081   US    TX
    ## 18:       1       7.0            1        NA            9  43.04672   US    NM
    ## 19:       1      25.0            1    1009.2            1  74.59676   US    AR
    ## 20:       5      19.5            5        NA            9  46.72222   US    OK
    ## 21:       5      14.0            5        NA            9  47.59617   US    AZ
    ## 22:       5      20.0            5    1016.3            5  66.76005   US    MD
    ## 23:       5      24.0            5    1016.5            5  88.77636   US    DE
    ## 24:       1      21.7            1    1013.6            1  76.78892   US    NJ
    ## 25:       1      12.8            1    1015.3            1  72.70817   US    IL
    ## 26:       5      20.6            5    1021.9            5  96.98682   US    MO
    ## 27:       5      21.1            5    1014.4            5  79.02548   US    KS
    ## 28:       1      12.8            1    1015.0            1  48.45233   US    UT
    ## 29:       1       6.7            1    1014.3            1  69.34826   US    NV
    ## 30:       5       7.2            5    1010.5            5  17.85877   US    CA
    ## 31:       1      24.8            1        NA            9 100.00000   US    MS
    ## 32:       5      20.0            5    1016.3            5  81.78746   US    MA
    ## 33:       5      18.0            5        NA            9  65.20835   US    RI
    ## 34:       5      20.6            5    1016.6            5  69.27841   US    PA
    ## 35:       1      15.6            1    1011.7            1  68.39318   US    NY
    ## 36:       5       9.1            5        NA            9  42.02597   US    MI
    ## 37:       5      18.0            5        NA            9  93.96867   US    IA
    ## 38:       5      19.1            5        NA            9  63.50739   US    NE
    ## 39:       5      10.6            5    1014.1            5  40.41716   US    OR
    ## 40:       5      20.6            5    1015.8            5  96.98682   US    NH
    ## 41:       1      10.0            1    1020.4            1  40.26115   US    VT
    ## 42:       5      16.7            5    1016.3            5  71.13935   US    ME
    ## 43:       1      12.0            1        NA            9  56.56040   US    WI
    ## 44:       5      24.4            5        NA            9  89.85960   US    SD
    ## 45:       1      16.0            1        NA            9  57.46078   US    MN
    ## 46:       5       6.1            5    1014.0            5  26.09109   US    WY
    ## 47:       1       3.3            1    1015.7            1  27.14411   US    MT
    ## 48:       5       1.1            5    1007.8            5  10.87160   US    ID
    ##     temp.qc dew.point dew.point.qc atm.press atm.press.qc        rh CTRY STATE
    ##     lat_mid  lon_mid   distance minrecord      n
    ##  1:  38.885  -80.400 0.17059015         1      1
    ##  2:  47.104 -122.416 0.12900000      4490   4490
    ##  3:  37.591  -84.672 0.09885848      5632   5632
    ##  4:  30.521  -91.983 0.12175796      3734   3734
    ##  5:  37.321  -77.558 0.08900562      8878   8878
    ##  6:  41.384  -72.682 0.17600000         1      1
    ##  7:  34.181  -80.634 0.12203688     14319  14319
    ##  8:  48.301  -99.621 0.41271055     22255  22255
    ##  9:  40.333  -83.078 0.06463745      9130   9130
    ## 10:  40.711  -86.296 0.07900000     13443  13443
    ## 11:  35.593  -86.246 0.21300000      2202   2202
    ## 12:  28.290  -81.876 0.28678215     15869  15869
    ## 13:  39.217 -105.861 0.38212694     28053  28053
    ## 14:  35.633  -79.101 0.04900000     38845  38845
    ## 15:  32.564  -83.270 0.28500000     75695  75695
    ## 16:  32.915  -86.557 0.34600578     41174  41174
    ## 17:  31.178  -97.691 0.03820995    202182 202182
    ## 18:  34.067 -105.990 0.75620169      7186   7186
    ## 19:  35.333  -92.767 0.33429328     21701  21701
    ## 20:  35.534  -97.350 0.12156480     64769  64769
    ## 21:  34.257 -111.733 0.39400000     23282  23282
    ## 22:  38.981  -76.684 0.19200000     17079  17079
    ## 23:  39.133  -75.467 0.00000000         1      1
    ## 24:  40.277  -74.417 0.25225384      9639   9639
    ## 25:  40.520  -88.751 0.17051100     52166  52166
    ## 26:  38.583  -93.183 0.12100000     25141  25141
    ## 27:  38.329  -97.430 0.30355560     18787  18787
    ## 28:  38.427 -113.012 0.00000000      3385   3385
    ## 29:  39.300 -116.891 0.93067771      9202   9202
    ## 30:  36.780 -120.448 0.50903929     77356  77356
    ## 31:  33.433  -90.078 0.75497285     32009  32009
    ## 32:  42.098  -70.918 0.24473046      9854   9854
    ## 33:  41.597  -71.433 0.02100000      3347   3347
    ## 34:  40.435  -76.922 0.22927058     20890  20890
    ## 35:  42.241  -75.412 0.56890069     13543  13543
    ## 36:  43.067  -84.688 0.25500000     64504  64504
    ## 37:  41.717  -93.566 0.02600000     49304  49304
    ## 38:  41.189  -98.054 0.30143822     22309  22309
    ## 39:  42.600 -123.364 0.00000000      7538   7538
    ## 40:  43.278  -71.503 0.07300000         2      2
    ## 41:  44.567  -72.562 0.06105735      8559   8559
    ## 42:  44.316  -69.667 0.13000000      9913   9913
    ## 43:  44.614  -89.774 0.19581879     74739  74739
    ## 44:  44.045  -99.318 0.27800000     11422  11422
    ## 45:  45.097  -94.382 0.12500000     94852  94852
    ## 46:  42.796 -108.389 0.27673995     30734  30734
    ## 47:  45.788 -110.440 0.08935883      7148   7148
    ## 48:  43.650 -116.240 0.08300000     14329  14329
    ##     lat_mid  lon_mid   distance minrecord      n

``` r
#leaflet(met_stations) %>%
#  addProviderTiles('OpenStreetMap') %>%
#  addCircles(lat=~lat,lng=~lon)
```

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
