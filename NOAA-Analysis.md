---
title: "Analysis of Weather Types for Economic and Health Damage"
author: "Eric Scuccimarra"
date: "28 December 2017"
output: 
    html_document:
        keep_md: true
---



## Abstract

The purpose of this analysis is to determine which types of severe weather events have the greatest impact on population health, and which have the greatest economic impact. To accomplish this we will look at storm data collected by NOAA from 1950 to 2011.

## Data Loading and Preprocessing

We start by downloading the bzipped CSV file from the course website and loading the data in.


```r
download.file("https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2", "StormData.csv.bz2")
rawdata <- read.csv(bzfile("StormData.csv.bz2")) 
```

Next we will remove the variables which are not relevant to the analysis.


```r
data <- rawdata[, c("EVTYPE","FATALITIES","INJURIES","PROPDMG","PROPDMGEXP","CROPDMG","CROPDMGEXP")]
```

The property and crop damage values are expressed in terms of a character which indicates whether the number is in hundreds, thousands, millions, or billions. To set the damage figures to be in dollars, we will convert the data accordingly. Once this is done we can remove the unneeded columns.


```r
data$PROPDMGEXP <- toupper(data$PROPDMGEXP)
data$CROPDMGEXP <- toupper(data$CROPDMGEXP)

exp_translations <- data.frame(EXP = c("H", "K","M","B"), VALUE=c(100,1000,1000000,1000000000))
data <- merge(data, exp_translations, by.x = "PROPDMGEXP", by.y="EXP")
names(data)[names(data) == "VALUE"] <- "PROPDMGMOD"
data <- merge(data, exp_translations, by.x="CROPDMGEXP", by.y="EXP")
names(data)[names(data) == "VALUE"] <- "CROPDMGMOD"

data$PROPDMG <- data$PROPDMG * data$PROPDMGMOD
data$CROPDMG <- data$CROPDMG * data$CROPDMGMOD

data <- data[,-c(1,2,8,9)]
```

Finally we can combine the Property and Crop damage into one column since we are only concerned with total damage and the breakdown is not relevant to this analysis.


```r
data$ECONDMG <- data$PROPDMG + data$CROPDMG
data <- data[,-c(4,5)]
```

Many of the hurricanes and storms are referenced individually by name, so in order to gauge which types of events are most damaging I attempted to group the types into 27 groups. When in doubt, I erred on the side of making the groups overly broad rather than overly specific with the assumption that this would provide the best generalized results.


```r
data$EVTYPE <- toupper(data$EVTYPE)
data$EVTYPE[grepl("FLOOD", data$EVTYPE)] <- "FLOOD"
data$EVTYPE[grepl("FLD", data$EVTYPE)] <- "FLOOD"
data$EVTYPE[grepl("HURRICANE", data$EVTYPE)] <- "HURRICANE"
data$EVTYPE[grepl("THUNDERSTORM", data$EVTYPE)] <- "THUNDERSTORM"
data$EVTYPE[grepl("STORM SURGE", data$EVTYPE)] <- "THUNDERSTORM"
data$EVTYPE[grepl("TROPICAL DEPRESSION", data$EVTYPE)] <- "TROPICAL STORM"
data$EVTYPE[grepl("TROPICAL STORM", data$EVTYPE)] <- "TROPICAL STORM"
data$EVTYPE[grepl("TORNADO", data$EVTYPE)] <- "TORNADO"
data$EVTYPE[grepl("GUSTNADO", data$EVTYPE)] <- "WIND"
data$EVTYPE[grepl("WIND", data$EVTYPE)] <- "WIND"
data$EVTYPE[grepl("SMOKE", data$EVTYPE)] <- "FIRE"
data$EVTYPE[grepl("FIRE", data$EVTYPE)] <- "FIRE"
data$EVTYPE[grepl("RIP CURRENT", data$EVTYPE)] <- "TIDE"
data$EVTYPE[grepl("ASTRONOMICAL", data$EVTYPE)] <- "TIDE"
data$EVTYPE[grepl("HEAT", data$EVTYPE)] <- "HEAT"
data$EVTYPE[grepl("ICE", data$EVTYPE)] <- "ICE"
data$EVTYPE[grepl("ICY", data$EVTYPE)] <- "ICE"
data$EVTYPE[grepl("RAIN", data$EVTYPE)] <- "RAIN"
data$EVTYPE[grepl("HAIL", data$EVTYPE)] <- "WINTER STORM"
data$EVTYPE[grepl("SNOW", data$EVTYPE)] <- "WINTER STORM"
data$EVTYPE[grepl("WINTER", data$EVTYPE)] <- "WINTER STORM"
data$EVTYPE[grepl("BLIZZARD", data$EVTYPE)] <- "WINTER STORM"
data$EVTYPE[grepl("SLEET", data$EVTYPE)] <- "WINTER STORM"
data$EVTYPE[grepl("SURF", data$EVTYPE)] <- "SURF"
data$EVTYPE[grepl("FREEZE", data$EVTYPE)] <- "EXTREME COLD"
data$EVTYPE[grepl("FOG", data$EVTYPE)] <- "FOG"
data$EVTYPE[grepl("DUST", data$EVTYPE)] <- "DUST STORM"
data$EVTYPE <- as.factor(data$EVTYPE)
```

## Analysis

### Economic Damage to Crops and Property

First we will look at economic damage by storm type, considering both the mean damage and the total damage caused by each type of event. To do this I calculate the total and mean damage caused by each type of event and then merge the two into one data frame.


```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
econ_dmg_total <- aggregate(ECONDMG ~ EVTYPE, data=data, sum)
econ_dmg_mean <- aggregate(ECONDMG ~ EVTYPE, data=data, mean, na.rm=TRUE)
econ_summary <- merge(econ_dmg_total, econ_dmg_mean, by="EVTYPE")
names(econ_summary) <- c("EVTYPE","TOTAL","MEAN")
```

To keep the plots readable, I select the top 5 types of events by both total and mean damage, and then keep the 8 types of events which are in the union of the two sets of events.


```r
es_top_total <- econ_summary[order(desc(econ_summary$TOTAL))[1:5],]$EVTYPE
es_top_mean <- econ_summary[order(desc(econ_summary$MEAN))[1:5],]$EVTYPE
economic_events <- subset(econ_summary, econ_summary$EVTYPE %in% es_top_total | econ_summary$EVTYPE %in% es_top_mean)
```


### Population Health Damage

Next I perform a similar analysis on the health effects. I begin by creating two data frames summarizing the fatalities, injuries, and sum of the fatalities and injuries by event type, calculating both the sum and mean. Then I merge the two resulting data frames together.


```r
data$ECONDMG <- NULL
data$BOTH = data$FATALITIES + data$INJURIES
health_means <- data %>% group_by(EVTYPE) %>% summarize_all(mean)
health_sums <- data %>% group_by(EVTYPE) %>% summarize_all(sum)
health_summary <- merge(health_means,health_sums,by="EVTYPE",suffixes=c(".MEAN",".TOTAL"))
```

As with economic damage, in the interest of a readable plot I get the top four ranking events for total fatalities, total injuries, mean fatalities and mean injuries and subset the data to include only events in the union of the four highest ranking events. This leaves nine event types in the summary data set.



```r
total_fatalities <- as.character(health_summary[order(desc(health_summary$FATALITIES.TOTAL))[1:4],]$EVTYPE)
total_injuries <- as.character(health_summary[order(desc(health_summary$INJURIES.TOTAL))[1:4],]$EVTYPE)
mean_fatalities <- as.character(health_summary[order(desc(health_summary$FATALITIES.MEAN))[1:4],]$EVTYPE)
mean_injuries <- as.character(health_summary[order(desc(health_summary$INJURIES.MEAN))[1:4],]$EVTYPE)
top_events = unique(c(total_fatalities,total_injuries,mean_fatalities,mean_injuries))
top_health <- subset(health_summary, health_summary$EVTYPE %in% top_events)
top_health$EVTYPE <- as.factor(top_health$EVTYPE)
```

Finally I melt the data for ease of plotting the variables against each other.


```r
library(reshape2)
library(ggplot2)
library(gridExtra)
```

```
## 
## Attaching package: 'gridExtra'
```

```
## The following object is masked from 'package:dplyr':
## 
##     combine
```

```r
melted <- melt(top_health)
```

```
## Using EVTYPE as id variables
```


## Results

## Economic Costs

Now we can plot the events with the total and mean costs:

```r
par(mfrow=c(1,2))
barplot(economic_events$TOTAL/1000000, names=economic_events$EVTYPE,ylab="Total Damage (in millions)",col=economic_events$EVTYPE,legend.text=economic_events$EVTYPE)
barplot(economic_events$MEAN/1000000, names=economic_events$EVTYPE,ylab="Mean Damage (in millions)",col=economic_events$EVTYPE)
```

![](NOAA-Analysis_files/figure-html/ploteconsummary-1.png)<!-- -->

We see that floods cause by far greater total damage, followed by hurricanes, while in terms of mean damage hurricanes are by far the most destructive, followed by tsunamis. This is attributable to the fact that there are 36132 flood events in the data set, while only 109 hurricane events. Although on average a hurricane causes 93.10 times more economic damage than a flood, the much higher number of floods makes them the greatest contributer to economic damage.

### Human Costs

#### Total injuries and fatalities by event type:


```r
ggplot(subset(melted,variable=="FATALITIES.TOTAL" | variable=="INJURIES.TOTAL"), aes(EVTYPE, value, fill=factor(variable,labels=c("Fatalities","Injuries"))))+ geom_bar(stat="identity") + labs(y="Event Type",x="Total Injuries and Fatalities",fill="Type") + theme(legend.position="bottom")
```

![](NOAA-Analysis_files/figure-html/totalhealthplot-1.png)<!-- -->

We see that tornadoes cause the greatest total number of fatalities and injuries, followed by floods.

#### Mean injuries and fatalities by event type:


```r
ggplot(subset(melted,variable=="FATALITIES.MEAN" | variable=="INJURIES.MEAN"), aes(EVTYPE, value, fill=factor(variable, labels=c("Fatalities","Injuries"))))+ geom_bar(stat="identity") + labs(x="Event Type",y="Mean Injuries and Fatalities",fill="Type") + theme(legend.position="bottom")
```

![](NOAA-Analysis_files/figure-html/meanhealthplot-1.png)<!-- -->

As with economic effects, the average number of fatalities and injuries differ from the total. Again hurricanes cause the greatest number of injuries on average, however tsunamis cause a greater number of fatalities on average. This appears to be caused by one outlier in the data. Out of the 19 tsunamis, 18 of them caused one or fewer injuries or fatalities.

## Conclusions

As far as economic damage, floods cause the greatest total damage with hurricanes causing the greatest mean damage. While each flood does not cause a great deal of damage, the sheer number of them causes the damage made by each to add up. In contrast, a relatively small number of hurricanes each cause a massive amount of damage.

In terms of fatalities and injuries, we observe a similar pattern. Tornados cause the most total injuries and fatalities, followed by floods. In terms of average number of injuries and fatalities, tsunamis cause the greatest average number of fatalities, however this is attributable to one outlier, as the remaining tsunamis cause no more than one injury or fatality each. If we exclude this outlier from the analysis, hurricanes have the highest average fatalities and injuries. In the health effects we also see that, despite a low average number of injuries caused by floods, the number of them causes the total injuries to add up.






[1]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf "Data Description"
