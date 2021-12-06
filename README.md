# FIA_Midwest_Weather_Data
pacman::p_load(pacman, caret, lars, tidyverse, rio)

states<-c("CT","DE","IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","RI","VT","WI","WV")
for(i in 1:length(states)){
  download.file(paste("https://apps.fs.usda.gov/fia/datamart/CSV/",states[i],"_PLOT.csv",sep=""),paste("FIA_",states[i],"PLOT.csv"),sep="")
}

for(i in 1:length(states)){
  download.file(paste("https://apps.fs.usda.gov/fia/datamart/CSV/",states[i],"_TREE.csv",sep=""),paste("FIA_",states[i],"TREE.csv"),sep="")
}

TERRA_Clim_Vars<-c("aet","def","PDSI","pet","ppt","q","soil","srad","swe","tmax","tmin","vap","vpd","ws")
for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2016.nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2016.nc"),sep="",mode = "wb")
}

for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2017.nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2017.nc"),sep="",mode = "wb")
}

for(i in 1:length(TERRA_Clim_Vars)){
  download.file(paste("https://climate.northwestknowledge.net/TERRACLIMATE-DATA/TerraClimate_",TERRA_Clim_Vars[i],"_2018.nc",sep=""),paste("TERACLIM_",TERRA_Clim_Vars[i],"2018.nc"),sep="",mode = "wb")
}

states<-c("CT","DE","IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","RI","VT","WI","WV")
years<-c("2016","2017","2018")
for (i in 1:length(states)) {
  for (j in 1:length(years)){
    FIA_Tree<- import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Project/FIA_Tree/FIA_ ",states[i]," TREE.csv",sep=""))
    head(FIA_Tree)
    is.data.frame(FIA_Tree)
    
    FIA_Plot<- import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Project/FIA_PLOT/FIA_ ",states[i]," PLOT.csv",sep=""))
    head(FIA_Plot)
    is.data.frame(FIA_Plot)
    
    FIA_Plot2<-FIA_Plot[,c(1,9,12,13,14,20,21,22)]
    FIA_Tree2<-FIA_Tree[,c(2,5,8,15,16,17,18,20)]
    colnames(FIA_Plot2)[1]="CN/PLTCN match"
    colnames(FIA_Tree2)[1]="CN/PLTCN match"
    FIA_Tree_Coord<-merge(FIA_Tree2,FIA_Plot2)
    FIA_Tree_Coord_Year<-FIA_Tree_Coord[FIA_Tree_Coord$MEASYEAR==years[j],]
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Tree_Coords/FIA_Trees_",years[j],"_Coord",sep=""))
    write.csv(FIA_Tree_Coord_Year,paste("FIA_",states[i],"trees_",years[j],".csv",sep=""))
  }
}
#Some of the files in the code above are too large to run in 32-bit RStudio. Code will run fine if RStudio is updated or in 64-bit R. 
pacman::p_load(pacman, caret, lars, tidyverse, rio)
state_coord<-import("C:/Users/Swenson/Documents/R/Denise/test_data/State Coords - Sheet1.csv")
years=c("2016","2017","2018")

library(reshape2)
library(ncdf4)
for(i in 1:length(years)){
  for(j in 1:nrow(state_coord)){
    filename<-paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_",years[i],"/TERACLIM_ aet ",years[i],".nc",sep="")
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
    var_tm<-ncvar_get(var,"aet",start = c(lon_ind[1],lat_ind[1],5),count = c(length(lon_ind),length(lat_ind),1))
    dim(var_tm)
    var_tm2<-var_tm
    row.names(var_tm2)<-lon[lon_ind]
    colnames(var_tm2)<-lat[lat_ind]
    var_tmmelt<-melt(var_tm2,id.vars=lat[lat_ind])
    colnames(var_tmmelt)<-c("lon","lat","var")
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_Lat_Lon_Var/TERRA_Clim_Var_",years[i],"",sep=""))
    write.csv(var_tmmelt, paste("Var_",state_coord[j,5],"lat_lon_",years[i],".csv",sep=""))
  }
}


#Now created a clim file specific to each state per year

pacman::p_load(pacman, caret, lars, tidyverse, rio)
years<-c("2016","2017","2018")
states<-c("CT","DE","IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","RI","VT","WI","WV")

for(i in 1:length(states)){
  for(j in 1:length(years)){
    
    FIA_Tree_Coord_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/FIA_Tree_Coords/FIA_Trees_",years[j],"_Coord/FIA_",states[i],"trees_",years[j],".csv",sep=""))
    as.matrix(FIA_Tree_Coord_year)
    
    FIA_year<-as.matrix(FIA_Tree_Coord_year[,c(14,13)])
    FIA_year<-as.data.frame(FIA_year)
    Uni_FIA_year<-unique(FIA_year[,c("LON","LAT")],)
    as.character(Uni_FIA_year$LAT)
    as.character(Uni_FIA_year$LON)
    
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Uni_FIA_year/Uni_FIA_",years[j],"",sep=""))
    write.csv(Uni_FIA_year,paste("Uni_FIA_",states[i],"_",years[j],".csv",sep=""))
    
  }
}


states<-c("CT","DE","IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","RI","VT","WI","WV")
years<-c("2016","2017","2018")
for(i in 1:length(states)){
  for(j in 1:length(years)){
    
    var_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_Lat_Lon_Var/TERRA_Clim_Var_",years[j],"/Var_",states[i],"lat_lon_",years[j],".csv",sep=""))
    colnames(var_year)<-c("V1","LON","LAT","aet")
    as.matrix(var_year)
    Clim_year<-as.matrix(var_year[,c(2,3,4)])
    Clim_year<-as.data.frame(Clim_year)
    station<-paste(rep("station",nrow(Clim_year)),c(1:nrow(Clim_year)),sep="_")
    Clim_year<-cbind(Clim_year,station)
    Clim_year<-na.omit(Clim_year)
    as.character(Clim_year$LON)
    as.character(Clim_year$LAT)
    as.character(Clim_year$station)
    setwd(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/aet/TERRA_Clim_aet_",years[j],"",sep=""))
    write.csv(Clim_year,paste("TERRA_Clim_aet_",states[i],"_",years[j],".csv",sep=""))
  }
}

#you have now changed the appearence/size of the TERRA Clim data for the var aet for each year and state. 
#This can now be used to find the closest station to the FIA plots.


pacman::p_load(pacman, caret, lars, tidyverse, rio)
years<-c("2016","2017","2018")
states<-c("IA","IL","IN","MA","MD","ME","MI","MN","MO","NH","NJ","NY","OH","PA","VT","WI","WV")
for(i in 1:length(states)){
  for(j in 1:length(years)){
    Clim_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Clim_year/aet/TERRA_Clim_aet_",years[j],"/TERRA_Clim_aet_",states[i],"_",years[j],".csv",sep=""))
    Clim_year<-Clim_year[,c(2,3,4,5)]
    
    Uni_FIA_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Clim_var/Uni_FIA_year/Uni_FIA_",years[j],"/Uni_FIA_",states[i],"_",years[j],".csv",sep=""))
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
        setwd("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet")
        write.csv(Uni_FIA_year_var,paste("Uni_FIA_",states[i],"_aet_",years[j],".csv",sep=""))
      }
    }
  }
}

#you have now found the closest station to each plot per state and per year. This chunck of code takes a very long time to run.

my.files.Clim_FIA_2016<-list.files(path="C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet",pattern = "2016.csv")
my.files.Clim_FIA_2017<-list.files(path="C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet",pattern = "2017.csv")
my.files.Clim_FIA_2018<-list.files(path="C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet",pattern = "2018.csv")

#You have now created a list of all the files to be combined into one csv. This will be helpful for the next section of code...

datalist<-list()
for(i in 1:length(my.files.Clim_FIA_2016)){
  Clim_FIA_2016<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet/",my.files.Clim_FIA_2016[i],"",sep=""))
  datalist[[i]]<-Clim_FIA_2016
}
Clim_FIA_all_state_2016<-do.call(rbind,datalist)
Clim_FIA_all_state_2016<-as.matrix(Clim_FIA_all_state_2016[,c(2,3)])
write.csv(Clim_FIA_all_state_2016,"C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet/All_state_FIA_clim_year/All_state_FIA_clim_2016.csv")

#You have now created a CSV of each FIA and aet clim var per year


year=c("2016","2017","2018")
var=c("3","4","5")
for(i in 1:length(year)){
  for(j in 1:length(var)){
    Clim_var<-import(paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_Lat_Lon_Var/TERRA_Clim_Var_",years[i],"/Midwest_",var[j],"_lat_lon_",years[i],".csv",sep=""))
    Clim_year<-import(paste("C:/Users/Swenson/Documents/R/Denise/TERRA_Clim/TERRA_Clim_Lat_Lon_Var/TERRA_Clim_Var_",years[i],"/Midwest_aet_lat_lon_",years[i],".csv",sep=""))
    station<-paste(rep("station",nrow(Clim_year)),c(1:nrow(Clim_year)),sep="_")
    Clim_year<-cbind(Clim_year,station)
    as.character(Clim_year$station)
    Clim_year<-Clim_year[,c(1,2,4)]
    Clim_var<-merge(Clim_var,Clim_year,by=c("LON","LAT"))
    Clim_var<-Clim_var[,c(3,4)]
    Uni_FIA_year_var<-import(paste("C:/Users/Swenson/Documents/R/Denise/Uni_FIA_Year_Var/Uni_FIA_all_states_aet/All_state_FIA_clim_year/All_state_FIA_clim_",years[i],".csv",sep=""))
    Uni_FIA_year_var<-merge(Uni_FIA_year_var,Clim_var,by="station")
    setwd("C:/Users/Swenson/Documents/R/Denise/FIA_Clim_coords_var")
    write.csv(Uni_FIA_year_var,paste("FIA_Clim_coords_May_",years[j],".csv",sep=""))
  }
}

#code to add other vars after determining the closest station
