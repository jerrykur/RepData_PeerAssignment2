# The Impacts of Weather Events on Humans and Budgets
J K  

Synopsis
--------
This report analyzes US National Oceanic and Atmospheric Data (NOAA) data from 1950 to 2011 to answers two questions related to storm event data.

1. What are the top events that cause the most impact to people (death or injury)?  
2. What are the top events that cause the most financial impact (property and crop damage)? 

The analysis of the NOAA data shows that Excessive Heat, Tornados, and Flooding cause the most deaths and the most injuries.  Therefore expendutures to mitigate the impact of these events are a good investment.

Analysis of data with respect to Property and Crop damage shows two different events are critical.  With respect to Property Damage, Hurricane, Storm Surge, and Flood events damage.  However, these often can be found in a single event, the Hurricane.   With respect to Crop Damage, Drought by far causes the most Crop Damage.

Data Processing
---------------

The following dependencies exist for this report.

```r
library(printr)
library(stringr)
library(lubridate)
library(plyr)
library(dplyr)
library(ggplot2)
library(grid)
library(gridExtra)
```

Read in data from two sources. The file, *repdata_data_StormData.csv.bz2*, contains the NOAA data.  The file, *evTypeMappings.csv*, contains corrective remappings of Event Types in the NOAA data.  These remappings group Event Types into one the of the 48 current Event Types used by NOAA.

```r
options(stringsAsFactors = FALSE)
if (!file.exists("repdata_data_StormData.csv.bz2")) {
  download.file('http://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2', 'repdata_data_StormData.csv.bz2',method="auto")
}
stormData <- read.csv(bzfile("repdata_data_StormData.csv.bz2"))
evTypeRep <- read.csv("evTypeMappings.csv")
```
There are : 902297 rows of data in epdata_data_StormData.csv.bz2.
There are : 222 rows of data in evTypeMappings.csv.


Based on information about the data from NOAA, we filter the NOAA data to remove old incomplete records from before 1996, and to remove records that had not impact on determining financial damage or personal injury/death.  

```r
# Add an event date type column to make it easier to filter data by date.
stormData$BeginDate <- as.Date(stormData$BGN_DATE, "%m/%d/%Y")
#  drop questionable data, earlier than 1996
stormDataFrom96 <- stormData[stormData$BeginDate >= "1996-01-01",]
#  drop  rows that do not contain property or crop damage or fatalities or injuries 
sd <- stormDataFrom96[stormDataFrom96$PROPDMG>0 | stormDataFrom96$CROPDMG>0 | stormDataFrom96$FATALITIES > 0 | stormDataFrom96$INJURIES > 0,]
```

In [this post](https://class.coursera.org/repdata-012/forum/thread?thread_id=52#comment-257) in the class forums it was brought up that the property damage for one event was entered twice and the second entry has the value in Billions$ instead of Millions$. Looking at the data we see that the event is the Napa Flood that occured at the end of 2005.  The property and crop damage for the second day of the flood should be removed.  The code below clears the property and crop damage for this event.

```r
sd$PROPDMG[which(sd$COUNTYNAME=='NAPA' & sd$EVTYPE=='FLOOD' & sd$BeginDate=="2006-01-01")]<-0
sd$CROPDMG[which(sd$COUNTYNAME=='NAPA' & sd$EVTYPE=='FLOOD' & sd$BeginDate=="2006-01-01")]<-0
```

With this errant record nutralized we can calculate the property and crop damages in dollars stored in new columns.  Also, to facilitate looking at data in total cost we added a new total damages column.

```r
#   Create and load property damage total column
sd$PROPDMG_total <- ifelse(sd$PROPDMGEXP == 'M', sd$PROPDMG * 1000000, ifelse(sd$PROPDMGEXP == 'B', sd$PROPDMG * 1000000000, ifelse(sd$PROPDMGEXP == 'K', sd$PROPDMG * 1000,  sd$PROPDMG  ) ) )
#   Create and loads crop damage total column
sd$CROPDMG_total <- ifelse(sd$CROPDMGEXP == 'M', sd$CROPDMG * 1000000, ifelse(sd$CROPDMGEXP == 'B', sd$CROPDMG * 1000000000, ifelse(sd$CROPDMGEXP == 'K', sd$CROPDMG * 1000, sd$CROPDMG  ) ) )
# Total Damage of all type
sd$DMG_total <- sd$CROPDMG_total+ sd$PROPDMG_total
```

The Event Type field in the data used in the assignment has a number of anamolous entries and does not match well with the NOAA 48 recognized event types.  To correct this we remap event types to match the current NOAA event types.

```r
sd$EventType.New <- sd$EVTYPE
# update EVTYPEs to match with those listed in Code book
for (i in 1:nrow(evTypeRep) ) 
{
    sd[sd$EventType.New==evTypeRep[i, c("Original")],c("EventType.New")] <- evTypeRep[i, c("NewValue")]
}
```

Once we  fix the event types we can look at start summing the impacts by Event Type.  We can also extract the top 10 events for various impacts.

```r
# sum across the Event Types
sd.sum <- ddply(sd, .(EventType.New), numcolwise(sum))

# get the top ten impacts for people and crops
sd.sum.fatals <- sd.sum[order(sd.sum$FATALITIES,  decreasing = TRUE) [1:10], ]
sd.sum.injuries <- sd.sum[order(sd.sum$INJURIES,  decreasing = TRUE) [1:10], ]
sd.sum.PropDmg <-sd.sum[order(sd.sum$PROPDMG_total,  decreasing = TRUE) [1:10], ]
sd.sum.CropDmg <- sd.sum[order(sd.sum$CROPDMG_total,  decreasing = TRUE) [1:10], ]
sd.sum.TotalDmg <- sd.sum[order(sd.sum$DMG_total,  decreasing = TRUE) [1:10], ]
```

Results
-------
We look at the impact of weather event on two factors.  The first is the impact to humans.  By impact to humans were consider Deaths and Injuries caused by weather events.  These are displayed in the following Figure. 


```r
theme_set(theme_gray(base_size = 10))
# Injuries
ginj <- ggplot(sd.sum.injuries, aes(y =INJURIES, x = reorder(EventType.New, -INJURIES))) +
     geom_bar(stat = 'identity', fill = '#5a9dd2') + 
     scale_x_discrete(labels = function(x) str_wrap(x, width = 10)) +
     ylab('Total Injuries') + xlab('Event Type')
# Fatalities
 gfat <- ggplot(sd.sum.fatals, aes(y =FATALITIES, x = reorder(EventType.New, -FATALITIES))) +
     geom_bar(stat = 'identity', fill = '#5a9dd2') + 
     scale_x_discrete(labels = function(x) str_wrap(x, width = 10)) +
     ylab('Total fatalities') + xlab('Event Type')
#  Put the plot one top of each other
 humanFig <- arrangeGrob(gfat, ginj, nrow = 2,  main=textGrob("Figure 1. Human impacts by Event Type")) 
# Display the combined chart
 print(humanFig)
```

![](Project2_files/figure-html/human_impact_by_event-1.png) 

As the figure shows, Excessive Heat, Tornados, and Flooding cause the most deaths.  These are all events in which some level of preventive measures and planning can be taken.  Excessive heat can be mitigated by increasing the avaiabilty of cooling centers and ensuring the people are checking on their neighbors to verify they are OK.  Tornados can be handled by adding additional requirements for safe spaces where a person can go and by better technology to detect and provide addition warning time before a Tornado hits an area.  Flooding can be handled by having defined evacutation plans to get people out of the way of rising waters.

The figure also shows that since the top three causes of Injuries are the same as the top causes of deaths.  Therefore, the same changes that mitigate deaths would also mitigate the leading causes of Injuries.  

Analyzing the data with regard to damage considers two types of damage, Property and Crop. These are shown in the figure below in millions of dollars.


```r
# add column converting values to millions of dollars
dmgDivisor = 100000000   # divide by 1 million
sd.sum.PropDmg$PROPDMG_totalM <- sd.sum.PropDmg$PROPDMG_total/dmgDivisor
sd.sum.CropDmg$CROPDMG_totalM <- sd.sum.CropDmg$CROPDMG_total/dmgDivisor
# plot settings
theme_set(theme_gray(base_size = 10))
# Property Damage
gPropDmg <- ggplot(sd.sum.PropDmg, aes(y=PROPDMG_totalM, x = reorder(EventType.New, -PROPDMG_totalM))) +
     geom_bar(stat = 'identity', fill = '#5a9dd2') + 
     scale_x_discrete(labels = function(x) str_wrap(x, width = 10)) +
     ylab('Property Damage (M$)') + xlab('Event Type')
 # Cope Damage
gCropDmg <- ggplot(sd.sum.CropDmg, aes(y=CROPDMG_totalM, x = reorder(EventType.New, -CROPDMG_totalM))) +
     geom_bar(stat = 'identity', fill = '#5a9dd2') + 
     scale_x_discrete(labels = function(x) str_wrap(x, width = 10)) +
     ylab('Crop Damage (M$)') + xlab('Event Type')
# Put the damages on one figure
 damageFig <- arrangeGrob(gPropDmg, gCropDmg,  main=textGrob("Figure 2. Damage by Event Type"), nrow = 2) 
# output the plot
 print(damageFig)
```

![](Project2_files/figure-html/damage_by_event-1.png) 

The figure shows that Property Damage is Hurricane, Storm Surge, and Floods are the top 3 events that cause damage.  However, since all three of these are present in Hurricanes it is likely they are related events.  This backed up by additional analysis of the data which show the three event types are often reported to the same meterological events.  One major such event that greatly impacts the damage figures was Hurricane Katrina.

From a planning perspective this implies that prevent of damage as a result of Hurricanes is a reasonable course of action.  This may involve building codes that prevent building in certain areas, structural codes that require a structure to be elevated to allow water to pass under the structure, or combinations.

However, when looking at Crop Damage we see that the Drought greatly overshadows all of the events.  Drought is a very difficult event to mitigate.  Some changes to drought tolerant crops can be made, but these must ensure that the new crop is sellable/desirable.  Irrigation can also help provided amply water supplies are available, but in many cases, such as the current West Coast droughts this is no longer the case.  Long term drought planning may require additional expensive steps such as building desalnization plants or additional resevoirs.  


Appendix
--------
In correcting the event types in the raw data a mapping file was used to translate the recorded event type to one of the NOAA defined event types.  To ensure that readers can reproduct the analysis above, the contents of the mapping file, *evTypeMappings.csv*, is displayed below.  The contents are displayed in raw format so a reader can be easily reproduce the mapping file by using a text editor to copy and paste the data into a local file named *evTypeMappings.csv*.

```bash
  cat evTypeMappings.csv
```

```
Original, NewValue
AGRICULTURAL FREEZE,Frost / Freeze
ASTRONOMICAL HIGH TIDE,Marine High Tide
ASTRONOMICAL LOW TIDE,Astronomical Low Tide
AVALANCHE,Avalanche
Beach Erosion,Storm Surge / Tide
BLACK ICE,Ice Storm
BLIZZARD,Blizzard
BLOWING DUST,Dust Storm
blowing snow,Blizzard
BRUSH FIRE,Wildfire
COASTAL  FLOODING/EROSION,Coastal Flood
COASTAL EROSION,Coastal Flood
COASTAL FLOOD,Coastal Flood
Coastal Flood,Coastal Flood
Coastal Flooding,Coastal Flood
COASTAL FLOODING,Coastal Flood
COASTAL FLOODING/EROSION,Coastal Flood
COASTAL STORM,Storm Surge / Tide
Coastal Storm,Storm Surge / Tide
COASTALSTORM,Storm Surge / Tide
COLD,Cold / Wind Chill
Cold,Cold / Wind Chill
COLD AND SNOW,Cold / Wind Chill
Cold Temperature,Cold / Wind Chill
COLD WEATHER,Cold / Wind Chill
COLD/WIND CHILL,Cold / Wind Chill
DAM BREAK,Flood
DAMAGING FREEZE,Frost / Freeze
Damaging Freeze,Frost / Freeze
DENSE FOG,Dense Fog
DENSE SMOKE,Dense Smoke
DOWNBURST,Heavy Rain
DROUGHT,Drought
DROWNING,Storm Surge / Tide
DRY MICROBURST,Heavy Wind
DUST DEVIL,Dust Devil
Dust Devil,Dust Devil
DUST STORM,Dust Storm
Early Frost,Frost / Freeze
Erosion/Cstl Flood,Coastal Flood
EXCESSIVE HEAT,Excessive Heat
EXCESSIVE SNOW,Winter Storm
Extended Cold,Cold / Wind Chill
EXTREME COLD,Cold / Wind Chill
Extreme Cold,Cold / Wind Chill
EXTREME COLD/WIND CHILL,Cold / Wind Chill
EXTREME WINDCHILL,Cold / Wind Chill
FALLING SNOW/ICE,Ice Storm
FLASH FLOOD,Flood
 FLASH FLOOD,Flood
FLASH FLOOD/FLOOD,Flood
FLOOD,Flood
FLOOD/FLASH/FLOOD,Flood
FOG,Dense Fog
Freeze,Frost / Freeze
FREEZE,Frost / Freeze
Freezing Drizzle,Ice Storm
FREEZING DRIZZLE,Ice Storm
Freezing drizzle,Ice Storm
FREEZING FOG,Freezing Fog
FREEZING RAIN,Ice Storm
Freezing Rain,Ice Storm
Freezing Spray,Ice Storm
FROST,Frost / Freeze
Frost/Freeze,Frost / Freeze
FROST/FREEZE,Frost / Freeze
FUNNEL CLOUD,Tornado
Glaze,Ice Storm
GLAZE,Ice Storm
GRADIENT WIND,Heavy Wind
gradient wind,Heavy Wind
Gradient wind,Heavy Wind
GUSTY WIND,Heavy Wind
GUSTY WIND/HAIL,Heavy Wind
GUSTY WIND/HVY RAIN,Heavy Wind
Gusty wind/rain,Heavy Wind
Gusty Winds,Heavy Wind
Gusty winds,Heavy Wind
GUSTY WINDS,Heavy Wind
HAIL,Hail
HARD FREEZE,Frost / Freeze
HAZARDOUS SURF,Storm Surge / Tide
HEAT,Excessive Heat
Heat Wave,Excessive Heat
HEAVY RAIN,Heavy Rain
Heavy Rain/High Surf,Marine Thunderstorm Wind
HEAVY SEAS,Marine Thunderstorm Wind
HEAVY SNOW,Heavy Snow
Heavy snow shower,Heavy Snow
Heavy Surf,Heavy Surf
HEAVY SURF,Heavy Surf
Heavy surf and wind,Heavy Surf
HEAVY SURF/HIGH SURF,Heavy Surf
HIGH SEAS,Marine Thunderstorm Wind
High Surf,Marine Thunderstorm Wind
HIGH SURF,Heavy Surf
   HIGH SURF ADVISORY,Heavy Surf
HIGH SWELLS,Heavy Surf
HIGH WATER,Flood
HIGH WIND,Strong Wind
HIGH WIND (G40),Strong Wind
HIGH WINDS,Strong Wind
HURRICANE,Hurricane / Typhoon
Hurricane Edouard,Hurricane / Typhoon
HURRICANE/TYPHOON,Hurricane / Typhoon
HYPERTHERMIA/EXPOSURE,Extreme Cold/Wind Chill
Hypothermia/Exposure,Extreme Cold/Wind Chill
HYPOTHERMIA/EXPOSURE,Extreme Cold/Wind Chill
Ice jam flood (minor,Debris Flow
ICE ON ROAD,Ice Storm
ICE ROADS,Ice Storm
ICE STORM,Ice Storm
ICY ROADS,Ice Storm
Lake Effect Snow,Lake-Effect Snow
LAKE EFFECT SNOW,Lake-Effect Snow
LAKE-EFFECT SNOW,Lake-Effect Snow
LAKESHORE FLOOD,Lakeshore Flooding
LANDSLIDE,Debris Flow
LANDSLIDES,Debris Flow
Landslump,Debris Flow
LANDSPOUT,Tornado
LATE SEASON SNOW,Winter Weather
LIGHT FREEZING RAIN,Heavy Rain
Light snow,Winter Weather
Light Snow,Winter Weather
LIGHT SNOW,Winter Weather
Light Snowfall,Winter Weather
LIGHTNING,Lightning
Marine Accident,Other
MARINE HAIL,Marine Hail
MARINE HIGH WIND,Marine Strong Wind
MARINE STRONG WIND,Marine Strong Wind
MARINE THUNDERSTORM WIND,Marine Thunderstorm Wind
MARINE TSTM WIND,Marine Thunderstorm Wind
Microburst,Strong Wind
MIXED PRECIP,Heavy Rain
Mixed Precipitation,Heavy Rain
MIXED PRECIPITATION,Heavy Rain
MUD SLIDE,Debris Flow
Mudslide,Debris Flow
MUDSLIDE,Debris Flow
Mudslides,Debris Flow
NON TSTM WIND,Heavy Wind
NON-SEVERE WIND DAMAGE,Heavy Wind
NON-TSTM WIND,Heavy Wind
Other,Other
OTHER,Other
RAIN,Heavy Rain
RAIN/SNOW,Heavy Rain
RECORD HEAT,Excessive Heat
RIP CURRENT,Rip Current
RIP CURRENTS,Rip Current
RIVER FLOOD,Flood
River Flooding,Flood
RIVER FLOODING,Flood
ROCK SLIDE,Debris Flow
ROGUE WAVE,Storm Surge / Tide
ROUGH SEAS,Storm Surge / Tide
ROUGH SURF,Storm Surge / Tide
SEICHE,Seiche
SMALL HAIL,Hail
SNOW,Heavy Snow
Snow,Heavy Snow
SNOW AND ICE,Heavy Snow
SNOW SQUALL,Heavy Snow
Snow Squalls,Heavy Snow
SNOW SQUALLS,Heavy Snow
STORM SURGE,Storm Surge / Tide
STORM SURGE/TIDE,Storm Surge / Tide
STRONG WIND,Strong Wind
Strong Wind,Strong Wind
Strong Winds,Strong Wind
STRONG WINDS,Strong Wind
THUNDERSTORM,Thunderstorm Wind
THUNDERSTORM WIND,Thunderstorm Wind
THUNDERSTORM WIND (G40),Thunderstorm Wind
TIDAL FLOODING,Flood
Tidal Flooding,Flood
TORNADO,Tornado
Torrential Rainfall,Heavy Rain
TROPICAL DEPRESSION,Tropical Depression
TROPICAL STORM,Tropical Storm
TSTM WIND,Thunderstorm Wind
 TSTM WIND,Thunderstorm Wind
Tstm Wind,Thunderstorm Wind
TSTM WIND  (G45),Thunderstorm Wind
TSTM WIND (41),Thunderstorm Wind
TSTM WIND (G35),Thunderstorm Wind
TSTM WIND (G40),Thunderstorm Wind
TSTM WIND (G45),Thunderstorm Wind
 TSTM WIND (G45),Thunderstorm Wind
TSTM WIND 40,Thunderstorm Wind
TSTM WIND 45,Thunderstorm Wind
TSTM WIND AND LIGHTNING,Thunderstorm Wind
TSTM WIND G45,Thunderstorm Wind
TSTM WIND/HAIL,Thunderstorm Wind
TSUNAMI,Tsunami
TYPHOON,Hurricane / Typhoon
Unseasonable Cold,Cold / Wind Chill
UNSEASONABLY COLD,Cold / Wind Chill
UNSEASONABLY WARM,Excessive Heat
UNSEASONAL RAIN,Heavy Rain
URBAN/SML STREAM FLD,Flood
VOLCANIC ASH,Volcanic Ash
WARM WEATHER,Excessive Heat
WATERSPOUT,Tornado
WET MICROBURST,Strong Wind
Whirlwind,Tornado
WHIRLWIND,Tornado
WILD/FOREST FIRE,Wildfire
WILDFIRE,Wildfire
Wind,Strong Wind
WIND,Strong Wind
WIND AND WAVE,Heavy Surf
Wind Damage,Strong Wind
WINDS,Strong Wind
WINTER STORM,Winter Storm
WINTER WEATHER,Winter Weather
WINTER WEATHER MIX,Winter Weather
WINTER WEATHER/MIX,Winter Weather
Wintry Mix,Winter Weather
WINTRY MIX,Winter Weather
```
