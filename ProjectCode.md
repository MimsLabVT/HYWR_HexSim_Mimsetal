Simulating the response of a threatened amphibian to climate-induced
reductions in breeding habitat
================

## Table of Contents

-   Introduction
-   Code for combination and manipulation of HexSim output files for
    downstream analyses
-   Code for analysis of derived HexSim output files

## Introduction and Caveats

This markdown includes code used to collate, shape, and analyze HexSim
output data associated with the manuscript: Simulating the response of a
threatened amphibian to climate-induced reductions in breeding habitat.
This manuscript focuses on an analysis of spatiotemporally dynamic
landscape wherein breeding habitat (ephemeral wetlands) blink in and out
for an at-risk, priority amphibian Arizona Treefrog, *Hyla wrightorum*,
in a disjunct and isolated subpopulation in the Huachuca Mountains in
southeastern Arizona.

Initially conducted with an older version of HexSim, the methods
required manually deriving individual landscape layers to simulate the
annual change in habitat availability based on historical water
availability. Therefore, there was many replicates required with each
annual landscape randomly assorted in the 100 year simulation period.
Thus, a form of environmental stochasticity was forced into across
modelling replicates.

As a result of the need of multiple replicates across landscape
iterations and for each sensitivity analysis and for each simulated
climate scenario, there was a bounty of data needed to be shaped and
combined to allow for downstream analyses. In the second section, we
present the code to replicate the analyses reported in the manuscript.

While we present these code for transparency of methodology and
potential replication, newer version of HexSim have added increased
functionality rendering some of the workarounds (which were required the
carpentry of the data we analyzed derived from the individually based
model and its iteration) not necessarily needed. Read the annotations to
better understand different aspects of the code and what was performed.
We would recommend these scripts being embedded into an R project for
convenience of directories working to your benefit.

Preferred Citation:

Mims, M., J.C. Drake, J.L. Lawler, & J.D. Olden. 2022. Simulating the
response of a threatened amphibian to climate-induced reductions in
breeding habitat. *Landscape Ecology* (In Review).

## 

``` r
## Code last updated: 20211012
## Written by Dr. Meryl Mims
## Edited by J. Drake 

## PACKAGES
library(dplyr)
library(reshape)

dirP <- "E"  # working directory addendum as needed
## CREATING SUMMARY CENSUS CSVs

## Census summaries

## Defining which Scenarios/sets/runs
scenario <- "A"   # A - H, referencing sensitivity analyses
set <- 1          # 
setmax <- 10      #
pondtest <- 6     #
pondtestmax <- 6  #
replicate <- 1    #
replicatemax <- 10#
censusnum <- 3 #0: with pond location, before reproduction; 
               #1: with pond location, after reproduction; 
               #2: after dispersal; 
               #3: after overwintering. Use 3 for reporting intergenerational trends.


## Create "platform" summary database to build with rbind
workspace <- paste(dirP,":HYWR_HexSim/HYWR_Scenario",
                   scenario,"/HYWR_Scenario",scenario,"_Set",set,
                   "/Results/HYWR_Scenario",scenario,"_",pondtest,".",
                   set,"/HYWR_Scenario",scenario,"_",pondtest,".",
                   set,"-[",replicate,"]",
                   "/HYWR_Scenario",scenario,"_",pondtest,".",set,".",censusnum,".csv",
                   sep="")

data <- read.csv(workspace)
identifier <- c(1) #use 1 for "platform"
data <- cbind(data, identifier) #adding unique identifier for each set; otherwise no way to distinguish
Summary <- data #finish creating platform as "Summary" database

#First for loop is to finish replicates within set 1; 
#otherwise first replicate is counted twice if 1:replicatemax
for (i in 2:replicatemax)
{
  workspace <- paste(dirP,":HYWR_HexSim/HYWR_Scenario",
                     scenario,"/HYWR_Scenario",scenario,"_Set",set,
                     "/Results/HYWR_Scenario",scenario,"_",pondtest,".",
                     set,"/HYWR_Scenario",scenario,"_",pondtest,".",
                     set,"-[",i,"]",
                     "/HYWR_Scenario",scenario,"_",pondtest,".",set,".",censusnum,".csv",
                     sep="")
  
  data <- read.csv(workspace)
  identifier <- c(1) #keep identifier at 1
  data <- cbind(data, identifier)
  Summary <- rbind(Summary, data)
}

## Nested for loops to cycle through set and replicates

for (i in 2:setmax)
{
  for (j in 1:replicatemax)
  {
    workspace <- paste(dirP,":HYWR_HexSim/HYWR_Scenario",
                       scenario,"/HYWR_Scenario",scenario,"_Set",i,
                       "/Results/HYWR_Scenario",scenario,"_",pondtest,".",
                       i,"/HYWR_Scenario",scenario,"_",pondtest,".",
                       i,"-[",j,"]",
                       "/HYWR_Scenario",scenario,"_",pondtest,".",i,".",censusnum,".csv",
                       sep="")

    data <- read.csv(workspace)
    identifier <- c(i) #set identifier to i to match set
    data <- cbind(data, identifier)
    Summary <- rbind(Summary, data)
  }
}

## Add unique group ID that combines identifier and Run
GroupID <- transform(Summary,GroupID=interaction(identifier,Run,sep="")) #Create unique ID for reps
GroupID <- dense_rank(GroupID$GroupID) #Assign only the unique ID to "GroupID", and rank from 1-100
Summary <- cbind(Summary,GroupID) #Add GroupID to Summary

## Export for downstream analyses
results <- paste("Results/HYWR_Scenario",
                 scenario,"/HYWR_Scenario",scenario,"_ResultsFullScenario",
                 sep="")
write.csv(Summary, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                 "_Census",censusnum,".csv",sep=''))


# run through iterations before starting next section (ponds X census)
```

``` r
## Read in for summarizing
scenario <- "A"
pondtest <- 6
censusnum <- 3
startgen <- 30 #keep at -1 until ready for subset. For subsets, uses all #'s greater than startgen (so 30 = start at 31;
#reflects start of "treatments" at gen 31.)

results <- paste("Results/HYWR_Scenario",
                 scenario,"/HYWR_Scenario",scenario,"_ResultsFullScenario",
                 sep="")

Summaryponds <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet",
                               pondtest,"_Census",censusnum,".csv",sep=''))

## Convert pond counts to presence-absence data
Summarybinary <- Summaryponds[,100:191] #make database with only pond columns, and only locations for group members
Summarybinary[Summarybinary > 0] <- 1 #convert to presence-absence
PondsOccupied <- rowSums(Summarybinary, na.rm=TRUE) #number of ponds occupied for each timestep
                                                    # na.rm=TRUE added to deal with C 

## Create new Summaryponds dataframe with GroupID, Time Step, and pond occupancy summed across ponds
Summaryoccupied <- data.frame(cbind(Summaryponds$GroupID, 
                                    Summaryponds$Time.Step, PondsOccupied))
colnames(Summaryoccupied) <- c("GroupID","Time.Step","PondsOccupied")

## Write to .csv - ponds occupied per timestep for all replicates
write.csv(Summaryoccupied, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                         "_Census",censusnum,"_SummaryOccupied.csv",sep=''), row.names=FALSE)

## Manipulate data for plots and analyses
OccPivot <- cast(Summaryoccupied, Time.Step ~ GroupID) #pivot table with Generation x replicate
OccPivotData <- OccPivot[,-1] #removes generation

OccMeans <- rowMeans(OccPivotData, na.rm=TRUE)  # na.rm=TRUE added to deal with C 
OccSDs <- apply(OccPivotData, 1, sd, na.rm=TRUE)# na.rm=TRUE added to deal with C 

OccforPlots <- data.frame(cbind(OccPivot$Time.Step, OccMeans, OccSDs)) #good to export for other analyses/tables

## Write to .csv - mean and sd # ponds occupied for each timestep (N=101)
write.csv(OccforPlots, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                                 "_Census",censusnum,"_PondsbyTimeStep_MeanandSD.csv",sep=''), row.names=FALSE)


## pivot by pond
PondOccCount <- data.frame(cbind(Summaryponds$GroupID, Summarybinary)) #create binary with gen & GroupID

## for subsetting to post-burnin
PondOccSubset <- data.frame(cbind(Summaryponds$GroupID, Summaryponds$Time.Step, Summarybinary))
PondOccSubset <- subset(PondOccSubset, Summaryponds.Time.Step > startgen)
PondOccCount <- PondOccSubset[,-2] #remove timestep

## set up without subsetting for now
PondRowNames <- c(1:91)
colnames(PondOccCount) <- c("Group","Matrix",PondRowNames)
PondOccByGen <- aggregate(. ~ Group, data=PondOccCount, FUN=sum) #Replicates, each pond
PondOccByGent <- data.frame(t(PondOccByGen[,-1]))
PondOccMean <- rowMeans(PondOccByGent, na.rm = TRUE) # na.rm=TRUE added to deal with C 
PondOccSDs <- apply(PondOccByGent, 1, sd, na.rm = TRUE) # na.rm=TRUE added to deal with C       
PondOccPercent <- 100*(PondOccMean/(100-(startgen)))
PondOccSDPer <- 100*(PondOccSDs/(100-(startgen)))
PondOccStats <- data.frame(cbind(PondOccMean, PondOccSDs, PondOccPercent, PondOccSDPer))

## Write to .csv - mean and sd # ponds occupied for each timestep (N=101)
write.csv(PondOccStats, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                             "_Census",censusnum,"_byPond_StartGen",startgen,"_TimeOccupied_Stats.csv",sep=''), row.names=TRUE)

# run through all pond X census combos before moving forward
```

``` r
## FOR POND SETS 4,5, AND 6 WITH NON-RECRUITING PONDS ####
## Sink pond info

## Read in for summarizing
scenario <- "A"
pondtest <- 6  # should this not be 4, 5, or 6? or do I just run all of them? -> run all i think
censusnumpre <- 0  # for this section should I run any other pre-post census combos?
censusnumpost <- 1
startgen <- 30 #keep at -1 until ready for subset. For subsets, uses all #'s greater than startgen (so 30 = start at 31;
               #reflects start of "treatments" at gen 31.)

results <- paste("Results/HYWR_Scenario",
                 scenario,"/HYWR_Scenario",scenario,"_ResultsFullScenario",
                 sep="")


PondsPre <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet",
                               pondtest,"_Census",censusnumpre,".csv",sep=''))
PondsPost <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet",
                           pondtest,"_Census",censusnumpost,".csv",sep=''))
PondsPre1 <- PondsPre[,8:99] #make database with only pond columns
PondsPost1 <- PondsPost[,8:99]

## Identify sink ponds using ratio of occupied to recruiting ponds (where recruiting ponds < 1, sinks = 1)
SinkPonds <- PondsPre1/PondsPost1
SinkPonds <- replace(SinkPonds, is.na(SinkPonds), 0)
SinkPonds[SinkPonds < 1] <- 0
write.csv(SinkPonds, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                              "_Sinks.csv",sep=''), row.names=FALSE)
SinkCounts <- rowSums(SinkPonds, na.rm=TRUE) #number of sinks for each timestep
                                             # na.rm=TRUE added to deal with C   

## Create new Summaryponds dataframe with GroupID, Time Step, and reproductive sinks

SinkbyTime <- data.frame(cbind(PondsPre$GroupID, 
                                    PondsPre$Time.Step, SinkCounts))
colnames(SinkbyTime) <- c("GroupID","Time.Step","Sinks")

## Write to .csv - sinks per timestep for all replicates
write.csv(SinkbyTime, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                                 "_SinksbyTime.csv",sep=''), row.names=FALSE)

## Manipulate data for plots and analyses
SinksTimePivot <- cast(SinkbyTime, Time.Step ~ GroupID) #pivot table with Generation x replicate
SinksTimePivotData <- SinksTimePivot[,-1] #removes generation
SinksTimeMeans <- rowMeans(SinksTimePivotData, na.rm=TRUE) # na.rm=TRUE added to deal with C 
SinksTimeSDs <- apply(SinksTimePivotData, 1, sd, na.rm=TRUE)# na.rm=TRUE added to deal with C 
SinksTimeSummary <- data.frame(cbind(SinksTimePivot$Time.Step, SinksTimeMeans, SinksTimeSDs)) #good to export for other analyses/tables

## Write to .csv - mean and sd # ponds occupied for each timestep (N=101)
write.csv(SinksTimeSummary, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                             "_SinksbyTime_MeanSD.csv",sep=''), row.names=FALSE)

## for subsetting to post-burnin
SinkbyPondSubset <- data.frame(cbind(PondsPre$GroupID, PondsPre$Time.Step, SinkPonds))
SinkbyPondSubset <- subset(SinkbyPondSubset, PondsPre.Time.Step > startgen)
SinkbyPond <- SinkbyPondSubset

## set up without subsetting for now
PondRowNames <- c(1:92)
colnames(SinkbyPond) <- c("Group","Generation",PondRowNames)
SinkbyPond <- SinkbyPond[,-2]
SinkByGen <- aggregate(. ~ Group, data=SinkbyPond, FUN=sum) #Replicates, each pond
SinkByGent <- data.frame(t(SinkByGen[,-1]))
SinkPondMean <- rowMeans(SinkByGent, na.rm=TRUE)   # na.rm=TRUE added to deal with C 
SinkPondSDs <- apply(SinkByGent, 1, sd, na.rm=TRUE)# na.rm=TRUE added to deal with C 
SinkPondPercent <- SinkPondMean/(101-(startgen+1))
SinkPondSDPer <- SinkPondSDs/(101-(startgen+1))
SinkPondStats <- data.frame(cbind(PondRowNames, SinkPondMean, SinkPondSDs, SinkPondPercent, SinkPondSDPer))

## Write to .csv - mean and sd # ponds occupied for each timestep (N=101)
write.csv(SinkPondStats, paste(results,"/","HYWR_Scenario",scenario,"PondSet",pondtest,
                              "_SinksbyPond_StartGen",startgen+1,".csv",sep=''), row.names=FALSE)

# run through each pond test/set (1-6)
```

## Downstream Analyses

This code represents some of the statistical analyses used to analyse
differences between climate based scenarios.

``` r
## Code last updated: 20211212

## Written by Dr. J. Drake - drakej@vt.edu

## This script is for down stream analyses of the HexSim output

# Base Scenario: Population Sizes -----------------------------------------

# Bring in data to the environment
# then add pondset # column for ID
# and then create a long data set of combined values
# Run ANOVA

nullfield <- NULL

results <- paste("Results/HYWR_Scenario",
                 scenario,"/HYWR_Scenario",scenario,"_ResultsPopSizeSum",
                 sep="")


scenario <- "A" # Main vs. sensitivity analyses
pondtest <- 1   # 1-6 'climate' driven habitat availability scenarios
startgen <- 21  # potentially run from 21, 41, 61, 81 depending on question asked


PopVal1 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet1_Census3_PopulationSize.csv",sep=''))
PopVal1$pondtest <- 1
PopVal2 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet2_Census3_PopulationSize.csv",sep=''))
PopVal2$pondtest <- 2
PopVal3 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet3_Census3_PopulationSize.csv",sep=''))
PopVal3$pondtest <- 3
PopVal4 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet4_Census3_PopulationSize.csv",sep=''))
PopVal4$pondtest <- 4
PopVal5 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet5_Census3_PopulationSize.csv",sep=''))
PopVal5$pondtest <- 5
PopVal6 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet6_Census3_PopulationSize.csv",sep=''))
PopVal6$pondtest <- 6

fullPopVals <- rbind(PopVal1,
                     PopVal2,
                     PopVal3,
                     PopVal4,
                     PopVal5,
                     PopVal6)

fullPopVals$pondtest <- as.factor(fullPopVals$pondtest)
# this particular analysis looks at all 100 iterations at gen 100 and compare btwn pondtests
Pop <- subset(fullPopVals, Generation > 99)
dim(Pop)
output <-lm(PopulationSize~pondtest, data=Pop)
tukeyoutput <- aov(PopulationSize~pondtest, data=Pop)
summary(output)
summary(tukeyoutput)
# tukey sig test
TukeyHSD(tukeyoutput)


## Repeat for last generation pond occupancy differences

OccVal1 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet1_Census1_PondOccupancy.csv",sep=''))
OccVal1$pondtest <- 1
OccVal2 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet2_Census1_PondOccupancy.csv",sep=''))
OccVal2$pondtest <- 2
OccVal3 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet3_Census1_PondOccupancy.csv",sep=''))
OccVal3$pondtest <- 3
OccVal4 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet4_Census1_PondOccupancy.csv",sep=''))
OccVal4$pondtest <- 4
OccVal5 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet5_Census1_PondOccupancy.csv",sep=''))
OccVal5$pondtest <- 5
OccVal6 <- read.csv(paste(results,"/","HYWR_Scenario",scenario,"PondSet6_Census1_PondOccupancy.csv",sep=''))
OccVal6$pondtest <- 6

fullOccVals <- rbind(OccVal1,
                     OccVal2,
                     OccVal3,
                     OccVal4,
                     OccVal5,
                     OccVal6)

fullOccVals$pondtest <- as.factor(fullOccVals$pondtest)
Occ <- subset(fullOccVals, Generation > 99)
dim(Occ)
output <-lm(PondsOccupied~pondtest, data=Occ)
tukeyoutput <- aov(PondsOccupied~pondtest, data=Occ)
summary(output)
summary(tukeyoutput)
# tukey sig test
TukeyHSD(tukeyoutput)
```

## Misc

Last updated: 20220720 Copyright: 2022 Creative Commons BY-NC-SA
license. No implied warranty or guarantee from use. Use at your own
discretion and risk.
