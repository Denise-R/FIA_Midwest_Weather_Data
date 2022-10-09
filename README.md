---
__Title: "FIA Midwest Climate Data"__

__Author: "Denise Rauschendorfer"__

__Date: "1/31/2022"__

This markdown file documents how data from three different publicly available databases ([FIA DataMart](https://apps.fs.usda.gov/fia/datamart/CSV/datamart_csv.html), [Climatology Lab](https://climate.northwestknowledge.net/TERRACLIMATE/index_directDownloads.php), and [USFS Insect & Disease Detection Survey](https://www.fs.usda.gov/foresthealth/applied-sciences/mapping-reporting/detection-surveys.shtml)) can be combined into one easily understandable dataframe. The final dataframe produced will match every tree recorded by the USFS in the northeastern portion of the United States with the climatology data recorded by the closest weather station to each individual tree. This will result in 14 different climatology variables per tree. Using ArcGIS 10.8.1 to extract the necessary Insect & Disease Detection Survey shapefiles provided by the USFS, each tree can then be related to the different types of insect and disease damage found nearby.

The following code was executed using R 4.2.1. 


## __Downloading Necessary Files__

### __Section 1: Downloading FIA Data__


This section of code downloads the <span style="color:green">"FIA_PLOT"</span> and <span style="color:green">"FIA_TREE"</span> data for every state in FIA region 9.  These files can be found on the [U.S. Forest Service Webpage](https://apps.fs.usda.gov/fia/datamart/CSV/datamart_csv.html).

![](https://www.fs.usda.gov/foresthealth/images/FS_regions.gif)


```{r eval=FALSE}
pacman::p_load(pacman, caret, lars, tidyverse, rio)

states<-c("CT","DE","IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","RI","VT","WI","WV")
for(i in 1:length(states)){
  download.file(paste("https://apps.fs.usda.gov/fia/datamart/CSV/",states[i],"_PLOT.csv",sep=""),paste("FIA_",states[i],"PLOT.csv"),sep="")
}

for(i in 1:length(states)){
  download.file(paste("https://apps.fs.usda.gov/fia/datamart/CSV/",states[i],"_TREE.csv",sep=""),paste("FIA_",states[i],"TREE.csv"),sep="")
}

```


### __Section 2: Downloading Climate Data__

This section of code downloads every climate variable reported in 2016-2018 from the [Climatology Lab](https://climate.northwestknowledge.net/TERRACLIMATE/index_directDownloads.php).

```{r eval=FALSE}
TERRA_Clim_Vars<-c("aet","def","PDSI","pet","ppt","q","soil","srad","swe","tmax","tmin","vap","vpd","ws")
for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2016. nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2016. nc"),sep="",mode = "wb")
}

for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2017. nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2017. nc"),sep="",mode = "wb")
}

for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2018. nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2018. nc"),sep="",mode = "wb")
}

```

***

## __Modifying the Appearance of Climate Data__

### __Section 3: Modifying Climatology Data__

The files downloaded from the [Climatology Lab](https://climate.northwestknowledge.net/TERRACLIMATE/index_directDownloads.php) in <span style="color:orange"> _Section 2_</span> are nc files that need to be opened, reduced in size, and converted into csv files. In order to accomplish this, the following steps are taken:

1. The file names (files downloaded in <span style="color:orange"> _Section 2_</span>) are entered and <span style="color:red">ncvar_open()</span> is used to create a connection to the file without actually opening it (<span style="color:blue">var</span>).
2. <span style="color:red">print (</span><span style="color:blue">var</span><span style="color:red">)</span> is used to view the types of data contained in each file. Within each file you can see the number of variables (1) and the dimensions of those variables (</span><span style="color:green">"lat"</span>, </span><span style="color:green">"lon"</span>, and </span><span style="color:green">"time"</span>). For example, in the file containing aet data from 2016 the size of </span><span style="color:green">"lat"</span> is 4320, </span><span style="color:green">"lon"</span> is 8640, and </span><span style="color:green">"time"</span> is 12.   Because this file is pretty big, we need to break them down into smaller sections.
3. <span style="color:red">ncvar_get ()</span> is used to request where/when the </span><span style="color:green">"lat"</span>, </span><span style="color:green">"lon"</span>, and </span><span style="color:green">"time"</span> data was recorded.
4. Because </span><span style="color:green">"time"</span> is recorded as days since 1900-01-01, it needs to be calibrated into a more easily understandable unit. To do this, <span style="color:red">as.date ()</span> is used to specify the origin and time zone.
5. To reduce the size of </span><span style="color:green">"lat"</span> and </span><span style="color:green">"lon"</span>, the latitudinal and longitudinal coordinates of each state are specified in the file </span><span style="color:green">"split_state_coords.csv"</span>. In this file, I further split each state into smaller sections to speed up the amount of time required to analyze the data.
*Note: if you get the error:“attempt to set 'colnames' on an object with less than two dimension”, the divisions you specified <span style="color:blue">lat_rng</span> or <span style="color:blue">lon_rng</span> are likely too small.*
6. <span style="color:red">ncvar_get ()</span> is used again to create a time series for each state per month.
7. The row and column names of the data frame are then changed to match the variables </span><span style="color:green">"lat"</span> and </span><span style="color:green">"lon"</span>.
8. <span style="color:red">melt()</span> is used to change the shape of the data frame and display </span><span style="color:green">"lat"</span>, </span><span style="color:green">"lon"</span>, and each corresponding climate variable in a more understandable way.
9. A column specifying which month each variable was pulled from is also added (</span><span style="color:green">"var_month"</span>).

This process is then repeated for every climate variable per month in each state section per year. 
The final csv files created each consist of 4 columns: latitude, longitude, a climate variable, and the month the data was collected.

```{r eval=FALSE}
years<-c("2016", "2017", "2018")
state_coord<-import("C:/Users/Swenson/Documents/R/Denise/Midwest_states_data/split_state_coords.csv")
month<-c(1:12)

library(reshape2)
library(ncdf4)
for(i in 1:length(years)){
  for(j in 1:nrow(state_coord)){
    for(k in 1:length(TERRA_Clim_Vars)){
      for(y in 1:length(month)){
        filename<-paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_",years[i],"/TERACLIM_ ",TERRA_Clim_Vars[k]," ",years[i],".nc",sep="")
        var<-nc_open(filename) 
        print(var)
        lat<-ncvar_get(var, "lat")
        lon<-ncvar_get(var, "lon")
        time<-ncvar_get(var, "time")
        time2<-as.Date(time, origin="1900-01-01", tz="UTC")
        lat_rng<-c(state_coord[j,3],state_coord[j,4])
        lon_rng<-c(state_coord[j,1],state_coord[j,2])
        lat_ind<-which(lat>=lat_rng[1]&lat<=lat_rng[2])
        lon_ind<-which(lon>=lon_rng[1]&lon<=lon_rng[2])
        lat[lat_ind]
        lon[lon_ind]
        var_tm<-ncvar_get(var,paste("",TERRA_Clim_Vars[k],"",sep=""),start = c(lon_ind[1],lat_ind[1],month[y]),count = c(length(lon_ind),length(lat_ind),1))
        dim(var_tm)
        var_tm2<-var_tm
        row.names(var_tm2)<-lon[lon_ind]
        colnames(var_tm2)<-lat[lat_ind]
        var_tmmelt<-melt(var_tm2,id.vars=lat[lat_ind])
        colnames(var_tmmelt)<-c("lon","lat",paste("",TERRA_Clim_Vars[k],"",sep=""))
        var_month<-matrix(paste("",month[y],"",sep=""),ncol=1,nrow=nrow(var_tmmelt))
        var_tmmelt<-cbind(var_tmmelt,var_month)
        setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_original/",month[y],"/",TERRA_Clim_Vars[k],"/",years[i],"",sep=""))
        write.csv(var_tmmelt, paste("",TERRA_Clim_Vars[k],"_",state_coord[j,5],"lat_lon_",years[i],".csv",sep=""))
      }
    }
  }
}
```

### __Section 4: Creating Climate Station Identifier__

This section of code adds a column (<span style="color:green">"station_ID"</span>) to each modified climate data file (files created in <span style="color:orange"> _Section 3_</span>) to identify the station the data was recorded. This is repeated for every climate variable per month in each state section per year.

```{r eval=FALSE}
Clim_states<-c("CT","DE","RI","MA","MD_1","MD_2","IA_1","IA_2","IA_3","IA_4","IA_5","IA_6","IA_7","IA_8","IA_9","IA_10","IA_11","IA_12","IA_13","IA_14","IA_15","IA_16","IN_1","IN_2","IN_3","IN_4","IN_5","IN_6","IN_7","IN_8","IN_9","IN_10","IN_11","IN_12","IN_13","IN_14","IN_15","IN_16","IL_1","IL_2","IL_3","IL_4","IL_5","IL_6","IL_7","IL_8","IL_9","IL_10","IL_11","IL_12","IL_13","IL_14","IL_15","IL_16","ME_1","ME_2","ME_3","ME_4","ME_5","ME_6","ME_7","ME_8","ME_9","ME_10","ME_11","ME_12","ME_13","ME_14","ME_15","ME_16","ME_17","ME_18","ME_19","ME_20","ME_21","ME_22","ME_23","ME_24","ME_25","ME_26","ME_27","ME_28","ME_29","ME_30","ME_31","ME_32","NH_1","NH_2","NH_3","NH_4","NH_5","NH_6","NH_7","NH_8","NH_9","NH_10","NH_11","NH_12","NH_13","NH_14","NH_15","NH_16","NJ_1","NJ_2","NJ_3","NJ_4","NJ_5","NJ_6","NJ_7","NJ_8","NJ_9","NJ_10","NJ_11","NJ_12","NJ_13","NJ_14","NJ_15","NJ_16","OH_1","OH_2","OH_3","OH_4","OH_5","OH_6","OH_7","OH_8","OH_9","OH_10","OH_11","OH_12","OH_13","OH_14","OH_15","OH_16","PA_1","PA_2","PA_3","PA_4","PA_5","PA_6","PA_7","PA_8","PA_9","PA_10","PA_11","PA_12","PA_13","PA_14","PA_15","PA_16","VT_1","VT_2","VT_3","VT_4","VT_5","VT_6","VT_7","VT_8","VT_9","VT_10","VT_11","VT_12","VT_13","VT_14","VT_15","VT_16","WV_1","WV_2","WV_3","WV_4","WV_5","WV_6","WV_7","WV_8","WV_9","WV_10","WV_11","WV_12","WV_13","WV_14","WV_15","WV_16","NY_1","NY_2","NY_3","NY_4","NY_5","NY_6","NY_7","NY_8","NY_9","NY_10","NY_11","NY_12","NY_13","NY_14","NY_15","NY_16","MO_1","MO_2","MO_3","MO_4","MO_5","MO_6","MO_7","MO_8","MO_9","MO_10","MO_11","MO_12","MO_13","MO_14","MO_15","MO_16","MI_1","MI_2","MI_3","MI_4","MI_5","MI_6","MI_7","MI_8","MI_9","MI_10","MI_11","MI_12","MI_13","MI_14","MI_15","MI_16","MI_17","MI_18","MI_19","MI_20","MI_21","MI_22","MI_23","MI_24","MI_25","MI_26","MI_27","MI_28","MI_29","MI_30","MI_31","MI_32","WI_1","WI_2","WI_3","WI_4","WI_5","WI_6","WI_7","WI_8","WI_9","WI_10","WI_11","WI_12","WI_13","WI_14","WI_15","WI_16","WI_17","WI_18","WI_19","WI_20","WI_21","WI_22","WI_23","WI_24","WI_25","WI_26","WI_27","WI_28","WI_29","WI_30","WI_31","WI_32","WI_33","WI_34","WI_35","WI_36","WI_37","WI_38","WI_39","WI_40","WI_41","WI_42","WI_43","WI_44","WI_45","WI_46","WI_47","WI_48","WI_49","WI_50","MN_1","MN_2","MN_3","MN_4","MN_5","MN_6","MN_7","MN_8","MN_9","MN_10","MN_11","MN_12","MN_13","MN_14","MN_15","MN_16","MN_17","MN_18","MN_19","MN_20","MN_21","MN_22","MN_23","MN_24","MN_25","MN_26","MN_27","MN_28","MN_29","MN_30","MN_31","MN_32","MN_33","MN_34","MN_35","MN_36","MN_37","MN_38","MN_39","MN_40","MN_41","MN_42","MN_43","MN_44","MN_45","MN_46","MN_47","MN_48","MN_49","MN_50")

for(i in 1:length(Clim_states)){
  for(j in 1:length(years)){
    for(y in 1:length(month)){
      for(k in 1:length(TERRA_Clim_Vars)){
        var_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_original/",month[y],"/",TERRA_Clim_Vars[k],"/",years[j],"/",TERRA_Clim_Vars[k],"_",Clim_states[i],"lat_lon_",years[j],".csv",sep=""))
        colnames(var_year)<-c("V1","LON","LAT",paste("",TERRA_Clim_Vars[k],"",sep=""),"var_month")
        as.matrix(var_year)
        Clim_year<-as.matrix(var_year[,c(2:5)])
        Clim_year<-as.data.frame(Clim_year)
        station<-paste(rep("station",nrow(Clim_year)),c(1:nrow(Clim_year)),sep="_")
        Clim_year<-cbind(Clim_year,station)
        Clim_year<-na.omit(Clim_year)
        as.character(Clim_year$LON)
        as.character(Clim_year$LAT)
        as.character(Clim_year$station)
        setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_reduced/",month[y],"/All_var/",years[j],"",sep=""))
        write.csv(Clim_year,paste("",TERRA_Clim_Vars[k],"_",Clim_states[i],"_",years[j],".csv",sep=""))
      }
    }
  }
}
```

***

## __Modifying the Appearance of FIA Data__

### __Section 5: Determining FIA Tree Coordinates__

This section of code combines the <span style="color:green">"FIA_PLOT"</span> and <span style="color:green">"FIA_TREE"</span> data obtained from the [U.S. Forest Service Webpage](https://apps.fs.usda.gov/fia/datamart/CSV/datamart_csv.html) (files downloaded in <span style="color:orange"> _Section 1_</span>). From the <span style="color:green">"FIA_PLOT"</span> data, we are interested in the variables <span style="color:green">"CN"</span>, <span style="color:green">"PLOT"</span>, <span style="color:green">"MEASYEAR"</span>, <span style="color:green">"MEASMON"</span>, <span style="color:green">"MEASDAY"</span>, <span style="color:green">"LAT"</span>, <span style="color:green">"LON"</span>, <span style="color:green">"ELEV"</span>. From the <span style="color:green">"FIA_TREE"</span> data, we are interested in the variables <span style="color:green">"PLTCN"</span>, <span style="color:green">"STATECD"</span>, <span style="color:green">"PLOT"</span>, <span style="color:green">"STATUSCD"</span>, <span style="color:green">"SPCD"</span>, <span style="color:green">"SPGRPCD"</span>, <span style="color:green">"DIA"</span>, <span style="color:green">"HT"</span>. In order to combine these files, the following steps are taken:

1. <span style="color:green">"FIA_TREE"</span> and <span style="color:green">"FIA_PLOT"</span> data is imported for each state per year (files downloaded in <span style="color:orange"> _Section 1_</span>).
2. The variables we are interested in are pulled out of each document.
3. Because the variables <span style="color:green">"CN"</span> and <span style="color:green">"PLTCN"</span> display the same data, these variables are renamed and used to merge the two documents.
4. All trees except ones sampled from 2016-2018 are removed from the data frame.

This process is repeated for each state per year creating csv files with <span style="color:green">"FIA_PLOT"</span> and <span style="color:green">"FIA_TREE"</span> variables.

A key for each column name can be found [here](https://www.fia.fs.fed.us/library/database-documentation/historic/ver3/FIADB_user%20manual_v3-0_p2_7_10_08. pdf).
A key for each species code (<span style="color:green">"SPCD"</span>) can be found [here](https://www.fia.fs.fed.us/program-features/urban/docs/Appendix%203a%20CORE%20Species%20List.pdf).

```{r eval=FALSE}

for (i in 1:length(states)) {
  for (j in 1:length(years)){
    FIA_Tree<- import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Project/FIA_Tree/FIA_ ",states[i]," TREE.csv",sep=""))
    FIA_Plot<- import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Project/FIA_PLOT/FIA_ ",states[i]," PLOT.csv",sep=""))
    
    FIA_Plot2<-FIA_Plot[,c(1,9,12,13,14,20,21,22)]
    FIA_Tree2<-FIA_Tree[,c(2,5,8,15,16,17,18,20)]
    colnames(FIA_Plot2)[1]="CN/PLTCN match"
    colnames(FIA_Tree2)[1]="CN/PLTCN match"
    FIA_Tree_Plot<-merge(FIA_Tree2,FIA_Plot2)
    FIA_Tree_Plot2<-FIA_Tree_Plot[FIA_Tree_Plot$MEASYEAR==years[j],]
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Tree_Coords/FIA_Trees_",years[j],"_Coord",sep=""))
    write.csv(FIA_Tree_Plot2,paste("FIA_",states[i],"trees_",years[j],".csv",sep=""))
  }
}
```

### __Section 6: Determining Unique FIA Coordinates__

The coordinates for each <span style="color:green">"FIA_TREE"</span> are recorded based on what <span style="color:green">"PLOT"</span> they are located in. As a result, multiple trees have the same coordinates. This section of code looks at the every merged <span style="color:green">"FIA_PLOT"</span> and <span style="color:green">"FIA_TREE"</span> csv file (files created in <span style="color:orange"> _Section 5_</span>) and pulls out every unique <span style="color:green">"LAT"</span>/<span style="color:green">"LON"</span> coordinate value. This process is repeated for every state per year.

```{r eval=FALSE}

for(i in 1:length(states)){
  for(j in 1:length(years)){
    
    FIA_Tree_Plot2<-import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Tree_Coords/FIA_Trees_",years[j],"_Coord/FIA_",states[i],"trees_",years[j],".csv",sep=""))
    as.matrix(FIA_Tree_Plot2)
    
    FIA_year<-as.matrix(FIA_Tree_Plot2[,c(14,13)])
    FIA_year<-as.data.frame(FIA_year)
    Uni_FIA_year<-unique(FIA_year[,c("LON","LAT")],)
    as.character(Uni_FIA_year$LAT)
    as.character(Uni_FIA_year$LON)
    
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Uni_FIA_year/Uni_FIA_",years[j],"",sep=""))
    write.csv(Uni_FIA_year,paste("Uni_FIA_",states[i],"_",years[j],".csv",sep=""))
    
  }
}
```

***

## __Relating Climate and FIA Data__

### __Section 7: Determining the Closest Climate Station to each FIA Plot__

For this section of code I chose to focus on the variable <span style="color:green">"aet"</span> in the month of May because the stations are the same for each variable per year; however, any month or variable would suffice. In order to accomplish this, the following steps are taken:

1. The reduced climate data files for each split state section per year in the month of May (a portion of the files created in <span style="color:orange"> _Section 4_</span>) along with the unique FIA state files (created in <span style="color:orange"> _Section 6_</span>) are imported.
2. The variables <span style="color:green">"V1"</span> and <span style="color:green">"var_month"</span> are removed from each imported climate file.
3. The variable <span style="color:green">"V1"</span> is removed from each imported FIA file.
4. A <span style="color:green">"unique ID"</span> for each plot location is added to each FIA file (data frame labeled <span style="color:blue">UNI_FIA_year</span>).
5. The distance between the coordinates of each unique FIA tree and the coordinates of each state's climate station is placed into a matrix (<span style="color:blue">var 1</span>) where each row is labeled by the respective <span style="color:green">"station ID"</span>.
6. The minimum distance between the coordinates of each unique FIA tree and the coordinates of each state's climate station is recorded in a matrix (<span style="color:blue">var 2</span>) with the respective <span style="color:green">"station ID"</span>.
7. The rows of the matrix <span style="color:blue">var 2</span> are changed to reflect each <span style="color:green">"unique ID"</span> they relate to, and the columns of <span style="color:blue">var 2</span> are renamed.
8. <span style="color:blue">var 2</span> and the modified unique FIA state files (<span style="color:blue">UNI_FIA_year</span>) are combined together to show the closest climate station to each unique FIA tree coordinate, along with the distance to that station.
9. All the variables in the reduced climate data files for each split state section per year in the month of May (<span style="color:blue">Clim_year</span>) are removed except for <span style="color:green">"aet"</span> (climate variable) and the <span style="color:green">"station_ID"</span> where data was recorded.
10. The climate variable <span style="color:green">"aet"</span> (or whatever variable you chose to complete this portion of code; data frame labeled <span style="color:blue">Clim_year_var</span>) is added to the data frame <span style="color:blue">Uni_FIA_year_var</span> based on the <span style="color:green">"station_ID"</span>.

The process is repeated for each split state section per year.

```{r eval=FALSE}

for(i in 1:length(Clim_states)){
  for(j in 1:length(years)){
    Clim_year<-read.csv(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_reduced/5/All_var/",years[j],"/aet_",Clim_states[i],"_",years[j],".csv",sep=""))
    Clim_year<-Clim_year[,c(2,3,4,6)]
    
    Uni_FIA_year<-read.csv(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Uni_FIA_year/Uni_FIA_",years[j],"/Uni_FIA_",gsub("_.*","",Clim_states[i]),"_",years[j],".csv",sep=""))
    Uni_FIA_year<-Uni_FIA_year[,c(2,3)]
    Unique_ID<-paste(rep("Unique ID",nrow(Uni_FIA_year)),c(1:nrow(Uni_FIA_year)),sep="_")
    Uni_FIA_year<-cbind(Uni_FIA_year,Unique_ID)
    as.character(Uni_FIA_year$Unique_ID)
    
    var1<-matrix(NA,ncol=1,nrow=nrow(Clim_year))
    var2<-matrix(NA,ncol=2,nrow=nrow(Uni_FIA_year))
    
    for(k in 1:nrow(Uni_FIA_year)){
      for(l in 1:nrow(Clim_year)){
        
        var1[l,] = dist(rbind(Uni_FIA_year[k,1:2], Clim_year[l,1:2]))
        rownames(var1)=Clim_year[,4]
        var2[k,1] = min(var1)
        var2[k,2] = rownames(var1)[rank(var1)==1]
        rownames(var2) = Uni_FIA_year[,3]
        colnames(var2)=c("Dist(km)", "station")
        Uni_FIA_year_var<-cbind(Uni_FIA_year,var2)
        Clim_year_var<-Clim_year[,c(3,4)]
        Uni_FIA_year_var<-merge(Uni_FIA_year_var,Clim_year_var,by="station")
        setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet_v2/",years[j],"",sep=""))
        write.csv(Uni_FIA_year_var,paste("Uni_FIA_",Clim_states[i],"_aet_",years[j],"_v2. csv",sep=""))
      }
    }
  }
}
```

### __Section 8: Adding Climate Station Coordinates__

For this section of code the coordinates of each closest climate station was added to each FIA file created in <span style="color:orange"> _Section 7_</span> . In order to accomplish this, the following steps are taken:

1. Files created in <span style="color:orange"> _Section 7_</span> (data frame labeled <span style="color:blue">Uni_FIA_Clim</span>) and each May aet file created in <span style="color:orange"> _Section 4_</span> (data frame labeled <span style="color:blue">aet</span>) are imported. Note that I choose the variable aet in the month of May because they were used to do the analysis in <span style="color:orange"> _Section 7_</span>.
2. All the columns of each May aet file were renamed to match <span style="color:blue">Uni_FIA_Clim</span> and the two data frames were merged together (<span style="color:blue">var</span>).
3. The variable <span style="color:green">"aet"</span>, <span style="color:green">"V1.x"</span>, <span style="color:green">"V1.y"</span>, and <span style="color:green">"var_month"</span> were all removed from the data frame <span style="color:blue">var</span>. I chose to remove <span style="color:green">"aet"</span> and <span style="color:green">"var_month"</span> because they will be added later; however, if you would like to keep these variables at this point in the analysis the code can easily be modified to do so.

This process was repeated for every state section per year.

```{r eval=FALSE}

for(i in 1:length(years)){
  for(j in 1:length(Clim_states)){
    Uni_FIA_Clim<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet_v2/",years[i],"/Uni_FIA_",Clim_states[j],"_aet_",years[i],"_v2. csv",sep=""))
    aet<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_reduced/5/All_var/",years[i],"/aet_",Clim_states[j],"_",years[i],".csv",sep=""))
    colnames(aet)<-c("V1","station_LON","station_LAT","aet","var_month","station")
    var<-merge(Uni_FIA_Clim,aet, by=c("station","aet"),na.omit=T)
    var<-var[,c(1,4:7,9:10)]
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state_sections/aet_only/",years[i],"",sep=""))
    write.csv(var, paste("Matched_aet",Clim_states[j],"_",years[i],".csv",sep=""))
  }
}
```

### __Section 9: Adding All variables__

This section of code adds every variable to each state section per month per year.To accomplish this, the following steps are taken:

1. Files created in <span style="color:orange"> _Section 4_</span> are listed by state section in <span style="color:blue">var_list</span> (14 docs per each list).
2. Each of the files in <span style="color:blue">var_list</span> are opened, and put the data into <span style="color:blue">datalist0</span>.
2. All of the data in <span style="color:blue">datalist0</span> is combined together into one data frame (<span style="color:blue">All_vars</span>). Now, <span style="color:blue">All_vars</span> contains every variable in each state section per month per year.
3. The <span style="color:green">"station_ID"</span> is then removed from the data frame and the station <span style="color:green">"LAT"</span>/<span style="color:green">"LON"</span> columns are renamed (data frame labeled <span style="color:blue">All_vars_2</span>).
4. The files created in <span style="color:orange"> _Section 8_</span> are imported (data frame labeled <span style="color:blue">var</span>).
5. The variable <span style="color:green">"V1"</span> is removed from the data frame <span style="color:blue">var</span>.
6. The data frames <span style="color:blue">var</span> and <span style="color:blue">All_vars_2</span> are combined together by the climate station coordinates. This adds the <span style="color:green">"unique_ID"</span>, FIA tree/plot coordinates (<span style="color:green">"LAT"</span>/<span style="color:green">"LON"</span>), and distance from each plot to the closest station (<span style="color:green">"Dist"</span>) to each file.

This process is repeated for each state section per month per year.

```{r eval=FALSE}
for(i in 1:length(years)){
  for(j in 1:length(Clim_states)){
    for(y in 1:length(month)){
      var_list<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_reduced/",month[y],"/All_var/",years[i],"",sep=""),
                           pattern=paste("_",Clim_states[j],"_",sep=""))
      datalist0<-list()
      for(k in 1:length(var_list)){
        var0<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/Split_var_reduced/",month[y],"/All_var/",years[i],"/",var_list[k],"",sep=""))
        datalist0[[k]]<-var0
      }
      All_vars<-Reduce(function(x, y) merge(x, y), datalist0)
      All_vars2<-All_vars[,c(2:4,6:19)]
      colnames(All_vars2)[1]<-"station_LON"
      colnames(All_vars2)[2]<-"station_LAT"
      var<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state_sections/aet_only/",years[i],"/Matched_aet",Clim_states[j],"_",years[i],".csv",sep=""))
      var<-var[,c(2:8)]
      All_vars3<-merge(var,All_vars2,by=c("station_LON","station_LAT"))
      setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state_sections/All_vars/",month[y],"/",years[i],"",sep=""))
      write.csv(All_vars3,paste("All_var_matched_",Clim_states[j],"_",years[i],".csv",sep=""))
    }
  }
}
```

***

## __Combining All Data per Year__

### __Section 10: Combining All State Sections per Year__

This section of code takes the files created in <span style="color:orange"> _Section 9_</span> and combines them into data frames per state per month per year. This is accomplished by the following steps:

1.  Files created in <span style="color:orange"> _Section 9_</span> (contains each variable per state section per month per year) are listed in <span style="color:blue">var1</span> by state, imported, placed into <span style="color:blue">datalist</span>, and bound together into the data frame <span style="color:blue">var 3</span>. 
2.  Only the closest station to each unique tree plot is kept and placed into the data frame <span style="color:blue">min_val</span>.This data is then placed into <span style="color:blue">datalist2</span> and bound together in <span style="color:blue">var 4</span>. 
3.  Files created in <span style="color:orange"> _Section 5_</span> (<span style="color:blue">FIA_tree_coord_Year</span>) that contain all FIA trees regardless of <span style="color:green">"LAT"</span>/<span style="color:green">"LON"</span> coordinates are uploaded and merged with <span style="color:blue">var 4</span> by the variables <span style="color:green">"LAT"</span> and <span style="color:green">"LON"</span> (data frame labeled <span style="color:blue">FIA_Full</span>).
4.  <span style="color:blue">FIA_Full</span> is then reduced in size to contain every variable except <span style="color:green">"V1x"</span> and <span style="color:green">"V1y"</span> (<span style="color:blue">var 5</span>).

This process is then repeated for each state section per month per year.

```{r eval=FALSE}
for(i in 1:length(years)){
  for(j in 1:length(states)){
    for(y in 1:length(month)){
      var1<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state_sections/All_vars/",month[y],"/",years[i],"",sep=""),
                       pattern=paste("All_var_matched_",states[j],"",sep=""))
      datalist<-list()
      for(k in 1:length(var1)){
        var2<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state_sections/All_vars/",month[y],"/",years[i],"/",var1[k],"",sep=""))
        datalist[[k]]<-var2
      }
      var3<-do.call(bind_rows,datalist)
      num<-1:nrow(var2)
      datalist2<-list()
      for(l in 1:length(num)){
        min_val<-min(var3[var3$Unique_ID==paste("Unique ID_",num[l],sep=""),]$Dist)
        datalist2[[l]]<-var3[grep(min_val,var3$Dist),]
      }
      var4<-do.call(bind_rows,datalist2)
      FIA_Tree_Coord_Year<-import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Tree_Coords/FIA_Trees_",years[i],"_Coord/FIA_",states[j],"trees_",years[i],".csv",sep=""))
      FIA_full<-merge(FIA_Tree_Coord_Year,var4,by=c("LAT","LON"))
      var5<-FIA_full[,c(1:2,4:15,17:36)]
      setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state/",month[y],"",sep=""))
      write.csv(var5,paste("All_var_matched_",states[j],"_",years[i],".csv",sep=""))
    }
  }
}
```

### __Section 11: Combining All States per Year__

This section of code takes the files created in <span style="color:orange"> _Section 10_</span> and combines them into data frames per month per year. This is accomplished by the following steps:

1.  Files created in <span style="color:orange"> _Section 10_</span> are listed in <span style="color:blue">var6</span> by state and imported (<span style="color:blue">var7</span>). 
2.  The variable <span style="color:green">"V1"</span> is removed from <span style="color:blue">var7</span> and the new <span style="color:blue">var7</span> is placed into <span style="color:blue">datalist3</span>. 
3.  All of the files in <span style="color:blue">datalist3</span> and bound together (<span style="color:blue">var8</span>).
4.  A file containing all of the common and scientific names of each tree species (<span style="color:green">"species_code.csv"</span>) is imported and merged with <span style="color:blue">var8</span>. 

This process is repeated for each state per month per year.

```{r eval=FALSE}
for(i in 1:length(years)){
  for(y in 1:length(month)){
    var6<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state/",month[y],"",sep=""),
                     pattern=paste("_",years[i],".csv",sep=""))
    datalist3<-list()
    for(k in 1:length(var6)){
      var7<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_state/",month[y],"/",var6[k],"",sep=""))
      var7<-var7[,c(2:35)]
      datalist3[[k]]<-var7
    }
    var8<-do.call(bind_rows,datalist3)
    state_codes<-import("C:/Users/Swenson/Documents/R/Denise/Midwest_states_data/species_codes.csv")
    var8<-merge(var8,state_codes)
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_year/",years[i],"",sep=""))
    write.csv(var8,paste("All_var_matched_",month[y],"_",years[i],".csv",sep=""))
  }
}
```

### __Section 12: Combining All Variables per Month per Year__

This section of code takes the files created in <span style="color:orange"> _Section 11_</span> and combines them into data frames per year. This is accomplished by the following steps:

1. The files created in <span style="color:orange"> _Section 11_</span> are listed in <span style="color:blue">var9</span> by year, imported (<span style="color:blue">var10</span>), and placed into <span style="color:blue">datalist4</span>. 
2. All of the data in <span style="color:blue">datalist4</span> is bound together in <span style="color:blue">var11</span>.
3. The variable <span style="color:green">"V1"</span> is removed from <span style="color:blue">var11</span>.  

This process is repeated for each year and creates three files (1 file per year).

Note: that this includes trees of all <span style="color:green">"STATUSCD"</span> (0:no status,1:live trees,2:dead trees,3:removed trees).

```{r eval=FALSE}
for(i in 1:length(years)){
  var9<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_year/",years[i],"",sep=""),
                   pattern=".csv")
  datalist4<-list()
  for(k in 1:length(var9)){
    var10<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_year/",years[i],"/",var9[k],"",sep=""))
    datalist4[[k]]<-var10
  }
  var11<-do.call(bind_rows,datalist4)
  var11<-var11[,c(2:35)]
  setwd("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_year_all_month")
  write.csv(var11,paste("All_var_matched_",years[i],".csv",sep=""))
}
```

***

## __Creating a Community Data Matrix (CDM)__

### __Section 13: Creating a CDM per Month per Year__

This section of code takes the files created in <span style="color:orange"> _Section 11_</span>, removes some variables, and creates a community data matrix (CDM) per month per year. This is accomplished by the following steps:

1. Files created in <span style="color:orange"> _Section 11_</span> are imported into a data frame labeled <span style="color:blue">var12</span> (need files that do not include all months because it complicates the CDM). 
2. The columns <span style="color:green">"STATECD"</span> AND <span style="color:green">"PLOT"</span> in <span style="color:blue">var12</span> are combined together to distinguish between plots labeled with the same number in different states (This will be necessary when creating the CDM).
3. The name of the column <span style="color:green">"PLOT"</span> in <span style="color:blue">var12</span> is changed to <span style="color:green">"STATECD_PLOT"</span>.
4. The data frame is then reduced to only include trees with a <span style="color:green">"STATUSCD"</span>=1 (status 1:live trees).
5. <span style="color:red">table ()</span> is used to determine how many times each tree species (specified by <span style="color:green">"Scientific Name"</span>) is found in each <span style="color:green">"STATECD_PLOT"</span> (<span style="color:blue">table_1</span>).
6. <span style="color:blue">table_1</span> is converted into a data frame and the names of the columns in <span style="color:blue">table_1</span> are changed to match <span style="color:blue">var12</span>.
7. The columns in <span style="color:blue">table_1</span> are ordered to match the structure required to run <span style="color:red">matrify()</span>, where the first column is the tree species, the second column is the state and plot where the tree was sampled, and the third column is the  number of times the tree species in column 1 was recorded in the state and plot in column 2. 
8. <span style="color:red">matrify()</span> is used to create a CDM from the data frame <span style="color:blue">table_1</span>. 
9. In order to relate <span style="color:blue">var13</span> to the variables in <span style="color:blue">var12</span>, all the unique <span style="color:green">"STATECD_PLOT"</span> variables from <span style="color:blue">var12</span> (or <span style="color:blue">table_1</span> for that matter) are put into there own data frame (<span style="color:blue">new_col</span>), labeled <span style="color:green">"STATECD_PLOT"</span>, and combined with data frame <span style="color:blue">var13</span> (<span style="color:blue">var14</span>).
10. The variables <span style="color:green">"V1"</span>, <span style="color:green">"SPCD"</span>, <span style="color:green">"SPGRPCD"</span>, <span style="color:green">"DIA"</span>, <span style="color:green">"HT"</span>, <span style="color:green">"common name"</span>, and <span style="color:green">"scientific name"</span> are removed because these variables do not apply to every tree within a specific <span style="color:green">"STATECD_PLOT"</span>.
11. Only unique rows in <span style="color:blue">var12</span> are kept. Notice that there is now only 1 row per <span style="color:green">"STATECD_PLOT"</span>.
12. <span style="color:blue">var14</span> and <span style="color:blue">var12</span> are combined into one data frame to create one CDM.

This process is then repeated for each month per year.

```{r eval=FALSE}
library(labdsv)
for(i in 1:length(years)){
  for(y in 1:length(month)){
    var12<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Matched_station/Per_year/",years[i],"/All_var_matched_",month[y],"_",years[i],".csv",sep=""))
    var12$PLOT<-paste(var12$STATECD,var12$PLOT,sep="_")
    colnames(var12)[colnames(var12)=="PLOT"]<-"STATECD_PLOT"
    var12<-var12[var12$STATUSCD==1,]
    table1<-table(var12$"Scientific Name",var12$"STATECD_PLOT")
    table1<-as.data.frame(table1)
    colnames(table1)<-c("Scientific Name","STATECD_PLOT","Freq")
    table1<-table1[,c(2,1,3)]
    var13<-matrify(table1)
    new_col<-as.data.frame(unique(var12$STATECD_PLOT))
    colnames(new_col)<-"STATECD_PLOT"
    var14<-cbind(new_col,var13)
    var12<-var12[,c(3:8,12:35)]
    var12<-unique(var12)
    var15<-merge(var14,var12)
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/CDM/Per_month/",years[i],"",sep=""))
    write.csv(var15,paste("CDM_",month[y],"_",years[i],".csv",sep=""))
  }
}
```

### __Section 14: Creating a CDM per Year__

This section of code takes the files created in <span style="color:orange"> _Section 13_</span> and creates a CDM per year. This is accomplished by the following steps:

1. The files created in <span style="color:orange"> _Section 13_</span> are listed in <span style="color:blue">var16</span> by year, imported (<span style="color:blue">var17</span>), and placed into <span style="color:blue">datalist5</span>. 
2. All of the data in <span style="color:blue">datalist5</span> is bound together in <span style="color:blue">var18</span>. 

This process is repeated for each year and creates three CDM files (1 file per year).

```{r eval=FALSE}
years<-c("2016","2017","2018")
for(i in 1:length(years)){
  var16<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/CDM/Per_month/",years[i],"",sep=""),
                    pattern=".csv")
  datalist5<-list()
  for(k in 1:length(var16)){
    var17<-import(paste("C:/Users/Swenson/Documents/R/Denise/CDM/Per_month/",years[i],"/",var16[k],"",sep=""))
    datalist5[[k]]<-var17
  }
  var18<-do.call(bind_rows,datalist5)
  setwd("C:/Users/Swenson/Documents/R/Denise/CDM/Per_year")
  write.csv(var18,paste("CDM_",years[i],".csv",sep=""))
}
```

### __Section 15: Reducing and Reorganizing each CDM File__

This section of code takes the files created in <span style="color:orange"> _Section 14_</span>, ensures each of these files has the same number of columns, reorganizes the data, and removes any unnecessary columns. This is accomplished by the following steps:

1. The 2016 file created in <span style="color:orange"> _Section 14_</span> is imported (<span style="color:blue">CDM_2016</span>).
2. The <span style="color:green">"V1"</span> columns are removed and the column <span style="color:green">"*Quercus nigra*"</span> is moved from the last column to the 51st column of the data frame. By moving the column <span style="color:green">"*Quercus nigra*"</span> all of the tree species are now in alphabetical order. 
3. A for loop is created to modify the 2017 and 2018 files created in <span style="color:orange"> _Section 14_</span> and the 2017 and 2018 files are imported (<span style="color:blue">CDM</span>).
4. A <span style="color:blue">"*Albizia_julibrissin*"</span> matrix is created with 1 column and the same number of rows as <span style="color:blue">CDM</span>, where every value in this matrix is NA.
5. The column name of the matrix <span style="color:blue">"*Albizia_julibrissin*"</span> is then changed to match the <span style="color:green">"*Albizia julibrissin*"</span> column in <span style="color:blue">CDM_2016</span>.
6. The matrix <span style="color:blue">"*Albizia_julibrissin*"</span> is then combined with <span style="color:blue">CDM</span>.
7. The columns of <span style="color:blue">CDM</span> are then reorganized and reduced in size to match <span style="color:blue">CDM_2016</span>.

This process creates three modified CDM files (1 file per year).

```{r eval=FALSE}
CDM_2016<-import("C:/Users/Swenson/Documents/R/Denise/CDM/Per_year/CDM_2016.csv")
CDM_2016<-CDM_2016[,c(3:52,95,53:94)]
setwd("C:/Users/Swenson/Documents/R/Denise/CDM/Per_year/Reorganized_data")
write.csv(CDM_2016,"CDM_2016_V2.csv")

years_17_18<-c("2017","2018")
for(i in 1:length(years_17_18)){
  CDM<-import(paste("C:/Users/Swenson/Documents/R/Denise/CDM/Per_year/CDM_",years_17_18[i],".csv",sep=""))
  Albizia_julibrissin<-matrix(NA,ncol=1,nrow=nrow(CDM))
  colnames(Albizia_julibrissin)<-"Albizia julibrissin"
  CDM<-cbind(CDM,Albizia_julibrissin)
  CDM<-CDM[,c(3:9,95,10:94)]
  setwd("C:/Users/Swenson/Documents/R/Denise/CDM/Per_year/Reorganized_data")
  write.csv(CDM,paste("CDM_",years_17_18[i],"_V2.csv",sep=""))
}
```

***

## __Reducing the Size of Files Created in Acrmap__

### __Section 16: CDM and USFS Surveyed Area__

This section of code reduces the number of columns in the CDM and USFS Disease Data surveyed area files created in Arcmap. In order to accomplish this, the following steps are taken:

1. The CDM and USFS Disease Data survey area text files are imported (<span style="color:blue">CDM_SA</span>).
2. Every column expect <span style="color:green">"TARGET_FID"</span>, <span style="color:green">"JOIN_FID"</span>, columns originating from the CDM files created in <span style="color:orange"> _Section 15_</span>, along with the <span style="color:green">"REGION_ID"</span>, <span style="color:green">"SURVEY_ID"</span>, <span style="color:green">"legacy"</span> data, and <span style="color:green">"source"</span> data from the USFS Disease Data file are removed from <span style="color:blue">CDM_SA</span>.

This process is repeated for each CDM and USFS surveyed area file per year.

```{r eval=FALSE}
for(i in 1:length(years)){
  CDM_SA<-import(paste("C:/Users/Swenson/Documents/R/Denise/GIS/Feature_Class_Tables/CDM_",years[i],"_Proj_Clip_SA_Join_Table.txt",sep=""))
  CDM_SA<-CDM_SA[,c(3,4,6:98,100,101,107:111,116,117)]
  setwd("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/CDM/Per_year")
  write.csv(CDM_SA,paste("CDM_",years[i],"_Proj_Clip_SA_Join_Table_Reduced.csv",sep=""))
}
```

### __Section 17: CDM and USFS Damaged Area__

This section of code reduces the number of columns in the CDM and USFS Disease Data damaged area files created in Arcmap. In order to accomplish this, the following steps are taken:

1. The CDM and USFS Disease Data damaged area text files are imported (<span style="color:blue">CDM_DA</span>).
2. Every column expect <span style="color:green">"TARGET_FID"</span>, <span style="color:green">"JOIN_FID"</span>, columns originating from the CDM files created in <span style="color:orange"> _Section 15_</span>, along with the <span style="color:green">"REGION_ID"</span>, <span style="color:green">"LABEL"</span>, <span style="color:green">"host"</span> columns, <span style="color:green">"DCA"</span> columns, <span style="color:green">"damage"</span> columns, columns that quantify damage, <span style="color:green">"NOTES"</span>, <span style="color:green">"OBSERVATION_COUNT"</span>, <span style="color:green">"ACRES"</span>, <span style="color:green">"SURVEY_ID"</span>, <span style="color:green">"legacy"</span> data, and <span style="color:green">"source"</span> data from the USFS Disease Data file are removed from <span style="color:blue">CDM_DA</span>.

This process is repeated for each CDM and USFS damaged area file per year.

```{r eval=FALSE}
for(i in 1:length(years)){
  CDM_DA<-import(paste("C:/Users/Swenson/Documents/R/Denise/GIS/Feature_Class_Tables/CDM_",years[i],"_Proj_Clip_DA_Join_Table.txt",sep=""))
  CDM_DA<-CDM_DA[,c(3,4,6:98,104:117,119,120,125,127:135,138,139)]
  setwd("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/CDM/Per_year")
  write.csv(CDM_DA,paste("CDM_",years[i],"_Proj_Clip_DA_Join_Table_Reduced.csv",sep=""))
}
```

### __Section 18: USFS Damaged Points and USFS Surveyed/Damaged Area__

This section of code reduces the number of columns in the USFS Disease Data damaged points and USFS surveyed/damaged area created in Arcmap. In order to accomplish this, the following steps are taken:

1. The USFS Disease Data damaged points and USFS surveyed/damaged area text file are imported (The <span style="color:blue">Dpts</span>).
2. Only the columns originating from the USFS Disease Data damaged points file, downloaded from the [USFS Webpage](https://www.fs.fed.us/foresthealth/applied-sciences/mapping-reporting/detection-surveys.shtml#idsdownloads), are preserved in <span style="color:blue">Dpts</span>.

This process is repeated for each USFS surveyed/damaged area (<span style="color:green">"DA"</span> and <span style="color:green">"SA"</span>) per year.

```{r eval=FALSE}
file_type<-c("SA","DA")
for(i in 1:length(years)){
  for(j in 1:length(file_type)){
    Dpts<-import(paste("C:/Users/Swenson/Documents/R/Denise/GIS/Feature_Class_Tables/Dpts_",years[i],"_Clip_",file_type[j],"_Join_Table.txt",sep=""))
    Dpts<-Dpts[,c(3,4,10,12:23,26:29,31,34:35)]
    setwd("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/Dpts/Per_year")
    write.csv(Dpts,paste("Dpts_",years[i],"_Clip_",file_type[j],"_Join_Table_Reduced.csv",sep=""))
  }
}
```

### __Section 19: USFS Damaged Points and USFS Surveyed/Damaged Area__

This section of code combines the files created in <span style="color:orange"> _Section 16_</span> and <span style="color:orange"> _Section 17_</span> into one csv file per USFS Disease Data survey/damage area. This is accomplished by the following steps:

1. The files created in <span style="color:orange"> _Section 16_</span> and <span style="color:orange"> _Section 17_</span> are listed in <span style="color:blue">var19</span> by USFS Disease Data type, imported (<span style="color:blue">var20</span>), and placed into <span style="color:blue">datalist6</span>. 
2. All of the data in <span style="color:blue">datalist6</span> is bound together in <span style="color:blue">var21</span>. 

This process is repeated for each USFS Disease Data survey/damage area.

```{r eval=FALSE}
for(i in 1:length(file_type)){
  var19<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/CDM/Per_year"),
                   pattern=paste("",file_type[i],"_Join_Table_Reduced.csv",sep=""))
  datalist6<-list()
  for(j in 1:length(var19)){
    var20<-import(paste("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/CDM/Per_year/",var19[j],"",sep=""))
    datalist6[[j]]<-var20
  }
  var21<-do.call(bind_rows,datalist6)
  setwd("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/CDM/All_years")
  write.csv(var21,paste("CDM_Proj_Clip_",file_type[i],"_Join_Table_Reduced.csv",sep=""))
}
```

### __Section 20: USFS Damaged Points and USFS Surveyed/Damaged Area__

This section of code combines the files created in <span style="color:orange"> _Section 18_</span> into one csv file per USFS Disease Data survey/damage area. This is accomplished by the following steps:

1. The files created in <span style="color:orange"> _Section 18_</span> are listed in <span style="color:blue">var22</span> by USFS Disease Data type, imported (<span style="color:blue">var23</span>), and placed into <span style="color:blue">datalist7</span>. 
2. All of the data in <span style="color:blue">datalist7</span> is bound together in <span style="color:blue">var24</span>. 

This process is repeated for each USFS Disease Data survey/damage area.

```{r eval=FALSE}
for(i in 1:length(file_type)){
  var22<-list.files(path=paste("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/Dpts/Per_year"),
                    pattern=paste("",file_type[i],"_Join_Table_Reduced.csv",sep=""))
  datalist7<-list()
  for(j in 1:length(var22)){
    var23<-import(paste("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/Dpts/Per_year/",var22[j],"",sep=""))
    datalist7[[j]]<-var23
  }
  var24<-do.call(bind_rows,datalist7)
  setwd("C:/Users/Swenson/Documents/R/Denise/GIS/Reduced_Join_Tables/Dpts/All_years")
  write.csv(var24,paste("DPts_Clip_",file_type[i],"_Join_Table_Reduced.csv",sep=""))
}
```

Last Updated: 10/5/2022
