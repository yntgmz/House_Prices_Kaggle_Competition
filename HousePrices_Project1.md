---
title: "House Prices: Advanced Regression Techniques"
author: "Yanet Gomez"
date: "2/19/2021"
github-repo: "https://github.com/yntgmz/House_Prices_Kaggle_Competition"
output:
  bookdown::gitbook:
    lib_dir: assets
  split_by: section
  config:
    toolbar:
      position: static
bookdown::pdf_book:
  keep_tex: true
bookdown::html_book:
  css: toc.css
new_session: true
bibliography: houseprices.bib

---

```{r setup, include=FALSE, eval =TRUE}
knitr::opts_chunk$set(echo = TRUE)

#Pacakges and libraries______________________________________
#install.packages("DataExplorer")  #for graphing missing data
#install.packages("data.table")
#install.packages("imputeMissings")
#install.packages("mice") #for RF prediction
#install.packages("vip") ##For variable imprtance in regularized regression
#install.packages("naniar") # plot missing values
#install.packages("mlr3pipelines")  
#install.packages("kableExtra")
#install.packages("ggridges")
library(bookdown)
library(kableExtra)
library(knitr)          #for tables
library(DataExplorer)   #for plot missing 
library(RColorBrewer)   #for color palattes
library(data.table)     #for custom function
library(ggplot2)        #for general plotting
library("GGally")       #for ggcor
library(Rcpp)           #missmap
library(Amelia)         #missmap
library(dplyr)          #mutate(), select()
library(ggridges)       #for ridge plot
library(stringr)        #for looking at columns containg "string"
library(imputeMissings) #impute missing values with median/mode or random forest
library(mice)           #impute missing values
library(forcats)        #for factor reorder
library(caret)          #for hot one encoding
library(plyr)           #to reorder factors 
library(vip)            #to show 
library(psych)          #corplot
require(Metrics)        #for rmse
library(DiagrammeR)     #for workflow diagram
library(gridExtra)      #for tables and graps
library(data.table)     #to create tables
library(RColorBrewer)   #For graph colors
library(naniar)         #plot missing values
library(car)            #for VIF
library(vip)            #for variable importance
library(gbm)            #for gbm model and plot
library(Metrics)        #for RMSE
library(Boruta)         #Boruta() for feature selection
library(gridExtra)
library(xgboost)
library(doParallel)
library(lattice)        #for plots
library(Matrix)
library(glmnet)         #fitting regularized models  
library(coefplot)       #for extracting coefficients

```



# Introduction

## Origin

* Semester Date: Spring 2021
* Class:  Capstone Portfolio (STAT 6382)
* Program: Master of Science in Data Analytics
* School: University of Houston Downtown

## Starting Point
House Prices: Advanced Regression Techniques is an ongoing Kaggle competition [@kagglehouseprices]. This "getting started" competition calls for extensive data pre-processing, feature engineering, and implementation of advanced regression techniques like random forest and gradient boosting. The data-set was made available through the Kaggle website at kaggle.com [@houseprices]. 

## Technical Skills

* Techniques: Data Cleaning, Exploratory Data Analysis, Feature Engineering, Advanced Regression Techniques, Machine Learning

* Program: R


## Problem Definition
Buying a house is a complicated decision making process for a lot of people. As Kaggle.com puts it: "Ask a home buyer to describe their dream house, and they probably won't begin with the height of the basement ceiling or the proximity to an east-west railroad" [@kagglehouseprices]. However, it is in fact, characteristics like these that influence the price of homes. Throughout this project we will explore a plethora of house features from kitchen quality to land slope, and evaluate their contribution to the sale price of homes. With some luck, or rather some data analytics skills, we will gain some insight into the real estate world of pricing residential homes. By understanding the relationship between key features and price, both buyers and sellers can optimize their price negotiations game, and make smarter financial decisions in the process of buying or selling a home. 

This project has three main goals. The first, as defined by Kaggle, is to predict the sales price value for each ID in the test set using advanced regression techniques. The second goal is to identify which features affect sale price the most. Lastly, I would like to explore which data analytics techniques lead to a better prediction in this case. 


* Which are the most relevant features that impact sales price?
* Which data pre-processing techniques improve the predictive value of the data?
* Which model performs best at predicting price?

 
## Data Description 
The data is the  "Ames Housing Dataset", provided at kaggle.com, which contains information about the sale of individual residential properties in Ames, Iowa from 2006 to 2010. The original dataset was compiled by Dean DeCock for use in data science education, and and is available online here: [Ames Housing Dataset](http://www.amstat.org/publications/jse/v19n3/decock/AmesHousing.xls). 

The data set contains 2919 observations and 80 explanatory variables, plus the variable of interest: Sales Price. Kaggle provided the data split into train and test sets. 


## Data Analytics Workflow
Here is an overview of the data analytics workflow for this project done with packages 'DiagrameR' [@diagramer] and 'webshot' [@webshot].


```{r Workflow, echo=FALSE, eval =TRUE, fig.cap="Workflow", fig.align = 'center', out.width="auto"}

grViz("
digraph boxes_and_circles {
graph [layout = dot, rankdir = LR]
node [shape = rectangle,color=AliceBlue, fontname = Helvetica,
     style = filled, fillcolor = AliceBlue, fontname=Helvetica, fontsize=14, fontcolor=SlateGray]
edge [color = SteelBlue,
     arrowsize = 1, penwidth=1.2]
rec1 [label = 'Data Collection']
rec2 [label = 'Data Cleaning']
rec3 [label =  'Data Transformation']
rec4 [label = 'Modeling']
rec5 [label = 'Prediction']
rec6 [label = 'Submission']


rec1-> rec2-> rec3-> rec4-> rec5-> rec6
}",
height = 100)

```
# Data
The data set contains 2919 observations and 80 explanatory variables, plus the variable of interest: Sales Price. Kaggle provided the data split into train and test sets. 

```{r RawDataImport, eval =TRUE, collapse = TRUE, comment="***"}
# Import Train and Tests Datasets
train_raw<-read.csv("train.csv",stringsAsFactors = TRUE) # 1460 rows,  81 columns
test_raw<-read.csv("test.csv", stringsAsFactors = TRUE)

#Merge Datasets for preprocessing steps. Separate into train and test sets later for modeling and prediction. 
test_raw$"SalePrice"<-0                   # Create "SalePrice" feature for test set
data_full<- rbind(train_raw, test_raw)    # Combine hdb_train and hdb_test

#Save ID column for submissiom
test_ID <-test_raw$Id

# Summarize Data Shape
cat("There are", dim(train_raw)[1], "rows and", dim(train_raw)[2], "columns in the Train data-set.\nThere are", dim(test_raw)[1], "rows and", dim(test_raw)[2], "columns in the Test data-set.\nThere are", dim(data_full)[1], "rows and", dim(data_full)[2], "columns in the combined data-set")

# Summarize Data Type
table(sapply(data_full, class)) 

```
Table 2.1. shows a snippet of the raw data including the first 10 rows and 10 columns plus SalesPrice.

```{r RawDataTable1, eval=TRUE, echo=FALSE}
raw_data_snip<-kable(data_full[1:10, c(1:10,81)], caption = "Raw Data Snippet",booktabs = TRUE)
raw_data_snip %>%
  kable_styling()%>%
  scroll_box(width = "100%", box_css = "border: 0px;")

```

# Data Pre-processing
The first step is to explore the data and identify any discrepancies that exist by examining the following: 

- Missing Values 

- Attribute types 

- Distribution, Skewness and Relationships

- Dimensionality 


## Identifying and Correcting Missing Values

### Detect Missing Values

```{r DetectMissVals, eval=TRUE, collapse = TRUE, comment="***"}
# Missing Values per column and rows 
col_na<-which(colSums(is.na(data_full)) > 0)
col_na_sum<-sum(colSums(is.na(data_full)) > 0)
cat("The total number of missing values in the combined data-set is:", sum(is.na(data_full)),"\nThe number of columns containing missing values is:", col_na_sum)
```

We can see from the heatmap in figure 3.1, that about 6% of the data is "missing". A simple strategy to deal with missing values is to eliminate the features or examples containing missing values. However, given that there are a total of 13965 missing values spread across 34 features, dropping these data points could mean loss of data critical to the analysis. Instead, we will deal with missing values by imputation using a variety of methods. 

```{r MissValsViz1, eval=TRUE, fig.cap= "Missing Values Heatmap", fig.align = 'center'}
# Heat map for missing values of the housing dataset with function missmap from library(Rcpp) 
missmap(data_full,col = c("blue","gray"), main ="Heatmap showing Missing values")
```

Figure 3.2., shows that the features "PoolQC", "MiscFeature", "Alley", and "Fence" have a high number of missing values. It might be tempting to drop these features, but with one quick look at the data description provided with the data [@de2011ames] we learn that "NA" in these cases means that the feature is not applicable, so it should be either "0" or "None".


```{r, MissValsViz2, eval=TRUE, fig.cap= "Number of Missing Values per Column", fig.align = 'center'}
NA_col_list <- sort(col_na, decreasing = T) # arrange the named list with descending order
gg_miss_var(data_full[,NA_col_list], show_pct = FALSE )+theme(text = element_text(size = 8,))
```


```{r MissValsSumm, eval=TRUE, collapse = TRUE}
# Summary of Missing Values per Column/Variable
kable(miss_var_summary(data_full[,NA_col_list]), caption='Summary of Missing Values per Column', booktabs = TRUE)

```

### Dealing With Missing Values

There are two types of missing values in this data-set, some values classified as "NA" are truly missing from the data, the information was not collected. The second type of "NA" means that the feature is not present, so "NA" in this case means either 'Zero' for numerical features,  'None' for categorical features, or 'No' for binary features. By examining the data and reading the data description provided [@de2011ames], we are able to determine each case. 

#### Manual Imputation of NAs due to Inconsistent Values

Starting with groups of related features for garage, basement, pool and masonry attributes to explore the columns containing missing values, we discover that there are a few inconsistencies in the raw data, for example for the feature MasVnrType, there is a missing value for row ID=2611, but the column "MasVnrArea" shows a value, this obviously indicates that there is a MasVnrType associated with this instance, so instead of replacing it with "None", we will impute the missing value with the most common value for this feature. 


```{r NAMasonry, eval=TRUE}
masonry_features <- names(data_full)[sapply(names(data_full), function(x) str_detect(x, "Mas"))]
mas_na<-which(is.na(data_full$MasVnrType) & data_full$MasVnrArea >0)
data_full[mas_na,masonry_features]
```

For features containing inconsistent values, listed in Table 2., we will impute the missing values mannually according to each case. 

```{r manual, echo=TRUE,eval=TRUE, collapse = TRUE}
Inconsitent_Values<- c("BsmtQual", "BsmtCond",  "GarageQual","GarageFinish", "GarageCond",  "GarageYrBlt","BsmtCond", "  PoolQC", "MasVnrArea")
Inconsitent_Values<-tibble("Feature"=Inconsitent_Values, "Method"=rep("Manual Imputation", length(Inconsitent_Values)))
kable(Inconsitent_Values, caption = "Inconsistent Values")
```
**Garage Features** 

_Row 2127 has a garage, given that it has values for "GarageArea" (360), "GarageType" (Detchd), and "GarageCars" (1), so fill in the 'GarageQual','GarageFinish', and 'GarageCond' with most the common  for those features values._

```{r Garage Features Missing Values, collapse = TRUE, eval=TRUE, echo=TRUE, comment="***"}
garage_features <- names(data_full)[sapply(names(data_full), function(x) str_detect(x, "Garage"))]
#View(data_full[which(is.na(data_full$GarageCond)), garage_features]) # To look at all garage features containing missing values
#count(data_full[which(is.na(data_full$GarageCond)), garage_features]) #159
kable(data_full[2127,garage_features])
#Get most commom values
kable(names(sapply(data_full[which( data_full$GarageCars == 1 & data_full$GarageType=="Detchd" ) ,garage_features], function(x) sort(table(x),  decreasing=TRUE)[1])), col.names = "Most Common Value per Feature") 
# Replace Values Manually
data_full[2127,'GarageQual']    = 'TA'
data_full[2127, 'GarageFinish'] = 'Unf'
data_full[2127, 'GarageCond']   = 'TA'
#Check changes
kable(data_full[2127,garage_features])
```

_Likewise, we can see that row 2577 has no garage, so we can fill in garage type with none._

```{r Garage Features Missing Values2, eval=TRUE, echo=TRUE, collapse = TRUE}
kable(data_full[2577,garage_features])
#Check and correct levels in Garage Type
levels(data_full$GarageType)
data_full$GarageType<-factor(data_full$GarageType, levels=c("None","2Types","Attchd",  "Basment","BuiltIn","CarPort","Detchd" ), ordered=FALSE)
#Update Garage type for row 2577
data_full[2577, 'GarageType'] = 'None'

```

_There is an error in GarageYrBlt, from the summary we see a year 2207. We will update to 2007._

```{r Garage Features Missing Values3, eval=TRUE, echo=TRUE, collapse = TRUE, comment="***"}
summary(data_full$GarageYrBlt)
kable(subset(data_full[garage_features], GarageYrBlt >= 2011))
data_full$GarageYrBlt[data_full$GarageYrBlt==2207] <- 2007
summary(data_full$GarageYrBlt) #Check changes
```

_For the missing values in GarageYrBlt, we will fill in NAs with with the year the house was built._

```{r Garage Year Built Missing Values, eval=TRUE, echo=TRUE,collapse = TRUE, comment="***"}
cat("GarageYrBlt has missing values:", sum(is.na(data_full$GarageYrBlt)))
# Fill in year garage built in the same year when house was built. ***
data_full$GarageYrBlt[is.na(data_full$GarageYrBlt)]<-data_full$YearBuilt[is.na(data_full$GarageYrBlt)]
cat("After inputing NA's with year YearBuilt values, GarageYrBlt has", sum(is.na(data_full$GarageYrBlt)), "missing values")

```
**Basement  Features**

_From viewing at all the basement columns we can see that basement condition is missing in rows 2041, 2186, 2525, and will replace with most the common feature for each column._

```{r BasementFeaturesNACorrected, eval=TRUE, echo=TRUE, collapse = TRUE}

basement_features <- names(data_full)[sapply(names(data_full), function(x) str_detect(x, "Bsmt"))]
#View(data_full[is.na(data_full$BsmtCond), basement_features]) 
data_full[c(2041, 2186,2525),basement_features]
#names(which.max(table(data_full$BsmtCond)))
data_full[c(2041,2186, 2525),'BsmtCond']=names(which.max(table(data_full$BsmtCond)))
#Check changes
data_full[c(2041, 2186,2525), basement_features]

```
**Pool Features**

_There are three pools where values for the 'quality' are missing, so we will fill in with the most common value based on the area of the pool._

```{r, PoolMissVals, echo=TRUE, eval=TRUE, collapse = TRUE}
pool_features <- names(data_full)[sapply(names(data_full), function(x) str_detect(x, "Pool"))]
data_full[c(2421,2504,2600),pool_features]
pool_na<-which(is.na(data_full$PoolQC) & data_full$PoolArea >0)
aggregate(data=data_full, PoolArea~PoolQC, mean, na.rm=TRUE)
data_full$PoolArea[which(is.na(data_full$PoolQC) & data_full$PoolArea >0)]
#Replace NA with most common values
data_full$PoolQC[data_full$Id == 2421] <- "Ex"
data_full$PoolQC[data_full$Id == 2504] <- "Ex"
data_full$PoolQC[data_full$Id == 2600] <- "Fa"
#Check changes
data_full[c(2421,2504,2600),pool_features]

```
***MasVnrType Features***

_There is a missing value for row ID=2611 in the MasVnrType column, we will replace with most common "MasVnrType"._ 

```{r, Mansory Features Missing Values, eval=TRUE, echo=TRUE, collapse = TRUE}
#Masonry Missing values_________________________________________________________________________
masonry_features <- names(data_full)[sapply(names(data_full), function(x) str_detect(x, "Mas"))]
mas_na<-which(is.na(data_full$MasVnrType) & data_full$MasVnrArea >0)
mas_na #ID=2611
data_full[2611,masonry_features]
data_full$MasVnrArea[which(is.na(data_full$MasVnrType) & data_full$MasVnrArea >0)]  #198
aggregate(data=data_full, MasVnrArea~MasVnrType, mean, na.rm=TRUE)
names(which.max(table(data_full$MasVnrType)))
summary(data_full$MasVnrType)  #most common is BrkFace
data_full$MasVnrType[data_full$Id == 2611] <- "BrkFace"
#Check Changes
data_full[2611,masonry_features]
```

#### Imputation with Mode for Categorical Features with a Few NAs
For some of the categorical features only missing a few values, we filled in the missing values with the most commonly occurring attribute value. Specially because for many of these the most frequent category was already over-represented, so it is pretty safe to assume that the missing values are more likely to be in the most common category. 

^[The graphs were produced with ggplot() function from package 'ggplot2' [@ggplot2].]

```{r plotcat, echo=FALSE, eval=TRUE, fig.cap="Categorical Features with Few NAs", fig.align = 'center',}

ut<-ggplot(data_full)+aes(Utilities)+ labs(title="Utilities")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))

func<-ggplot(data_full)+aes(Functional)+ labs(title="Functional")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))

elec<-ggplot(data_full)+aes(Electrical)+ labs(title="Electrical")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))

sale<-ggplot(data_full)+aes(SaleType)+ labs(title="SaleType")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))


kitch<-ggplot(data_full)+aes(KitchenQual)+ labs(title="KitchenQual")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))

ext1<-ggplot(data_full)+aes(Exterior1st)+ labs(title="Exterior1st")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))

ext2<-ggplot(data_full)+aes(Exterior2nd)+ labs(title="Exterior2nd")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))


grid.arrange(ut, func, elec, sale,
             ncol = 2, nrow = 2)

#grid.arrange(kitch, ext1, ext2,ncol = 2, nrow = 2)

```
```{r, HPFewMiss, eval=TRUE, echo=FALSE}
Few_Missing<-c("Utilities", "Functional", "Exterior1st", "Exterior2nd", "Electrical", "KitchenQual", "SaleType")
Few_Missing<-tibble("Feature"=Few_Missing, "Method"=rep("Mode", length(Few_Missing)))
kable(Few_Missing, caption = "Few NAs")
```


```{r CatMode, eval=TRUE}
#Replace with most common value since these are missing very few values: 
na_m <- c( "Utilities", "Functional", "Exterior1st", "Exterior2nd", "Electrical", "KitchenQual", "SaleType")
data_full[,na_m] <- apply(data_full[,na_m], 2,
                         function(x) {replace(x, is.na(x), names(which.max(table(x))))})

```

#### Imputation with "None" for Categorical Features 

According to the data description file [@de2011ames] for the categorical features listed in Table 5., NA means that the feature is not present in the house. For example, the house doesn't have a garage. For these features the NA values were replaced with "None". 

```{r, HPCat, eval=TRUE, echo=FALSE}
Categorical <- c("GarageFinish", "GarageQual", "GarageType", "GarageCond", "BsmtCond", "BsmtExposure", "BsmtQual", "BsmtFinType1", "BsmtFinType2", "FireplaceQu", "EnclosedPorch","MiscFeature","Alley", "Fence", "MasVnrType")
Categorical<-tibble("Feature"=Categorical, "Method"=rep("None", length(Categorical)))
kable(Categorical, caption = "Categorical Features where NA means 'None'")
```


```{r, Categorical Features Missing Values, eval=TRUE, echo=TRUE, collapse = TRUE}
#Replace with None, as in feature is not present:________________________________
na_none <- c("GarageFinish", "GarageQual", "GarageType", "GarageCond", "BsmtCond", "BsmtExposure", "BsmtQual", "BsmtFinType1", "BsmtFinType2", "FireplaceQu", "EnclosedPorch","PoolQC","MiscFeature","Alley", "Fence", "MasVnrType")

data_full[,na_none] <- apply(data_full[,na_none], 2,
                          function(x) {replace(x, is.na(x), "None")})

```
#### Imputation with Zero (0) for Numerical Features where NA means Feature is not Present
According to the data description file [@de2011ames] for the numerical features listed in Table 3.  NA means that the feature is not present in the house, so we replaced NA with the number zero for these features. 
```{r HPCont, eval=TRUE, echo=TRUE, collapse = TRUE}
Continous <- c("BsmtFinSF1","BsmtFinSF2", "BsmtUnfSF", "TotalBsmtSF", "BsmtFullBath", "BsmtHalfBath","GarageCars","GarageArea", "MasVnrArea")
Continous <- tibble("Feature"=Continous, "Method"=rep("0",length(Continous)))
kable(Continous, caption = "NA means '0'")
```


```{r, NumMissVals, eval=TRUE, echo=TRUE, collapse = TRUE}
#Replace with O, since these features are integers, and NA means there is zero _________
na_z <- c("BsmtFinSF1","BsmtFinSF2", "BsmtUnfSF", "TotalBsmtSF", "BsmtFullBath", "BsmtHalfBath","GarageCars","GarageArea", "MasVnrArea")

data_full[,na_z] <- apply(data_full[,na_z], 2,
                          function(x) {replace(x, is.na(x), 0)})
```


#### Imputation with estimation for Features with Large Number of Missing Values
Lastly, we were left with two features as seen in Figure 3.4. LotFrontage, which captures the linear feet of street-connected to the property; and MSZoning, which indicates the  different zoning classifications ranging from agricultural to residential. Because these features had a large percentage of missing values, it might be better to estimate their value based on other attributes. I used the mice package[@mice], which uses a Random Forest algorithm to estimate the missing values.


```{r RMissVals,eval=TRUE, fig.cap= "Remaining Missing Values", fig.align = 'center', echo=TRUE}
cols_missing_values2<-data_full[which(colSums(is.na(data_full))>0)]
plot_missing(cols_missing_values2)
```
^[The graph was produced with function plot_missing() from package 'DataExplorer' [@plotmissing].]

```{r ManyMiss,eval=TRUE, echo=FALSE}
Many_Missing<-c("LotFrontage", "MSZoning")
Many_Missing <- tibble("Feature"=Many_Missing, "Method"=rep("Prediction",length(Many_Missing)));
kable(Many_Missing,caption = "Large Number of Missing Values")

```

```{r MissValsMany, echo=TRUE, collapse = TRUE}
#Impute LotFrontage (This takes a long time to run)
library(mice)
set.seed(123)
mice_rf_mod<- mice(data_full[, !names(data_full) %in% c('Id', 'SalePrice')], method ='rf', printFlag = FALSE)
mice_output <- complete(mice_rf_mod)
#Inpute LotFrontage
sum(is.na(data_full$LotFrontage))
data_full$LotFrontage[is.na(data_full$LotFrontage)] <- mice_output$LotFrontage[is.na(data_full$LotFrontage)]
sum(is.na(data_full$LotFrontage))
#Inpute MSZoning
set.seed(123)
sum(is.na(data_full$MSZoning))
data_full$MSZoning[is.na(data_full$MSZoning)] <- mice_output$MSZoning[is.na(data_full$MSZoning)]
sum(is.na(data_full$MSZoning))
```
**After imputing values for all NA's, confirm all  missing values are now cleared.**

```{r MissValsFinal,eval=TRUE, fig.cap= "Final Missing Values HeatMap", echo=TRUE, fig.align = 'center', collapse = TRUE}
sum(is.na(data_full))           
missmap(data_full,col = c("red","gray"), main ="Heatmap showing Missing values")
```

## Correct Data Types
The next step in pre-processing, after all the NA values have been cleared, is to identify and correct data type inconsistencies.  The raw data shows that there are  43 factors, 37 integers, and 1 numeric data types. 

```{r data_type_raw, eval=TRUE,collapse = TRUE, echo=FALSE}
table(sapply(train_raw, class))    
table(sapply(test_raw, class))
table(sapply(data_full, class))
```

**Our current dataset looks like this:**

```{r data_type_combined, eval=TRUE,collapse = TRUE, echo=FALSE}

Integers <- which(lapply(data_full,class) == "integer")     # Integer features
cat("Integer Features (int):", names(Integers))

Factors <- which(lapply(data_full, class) == "factor")    # Categorical features
cat("Categorical Features:", names(Factors))

Characters <- which(lapply(data_full, class) == c("character"))    # Character features
cat("Character features:", names(Characters))

Numeric <- which(lapply(data_full, class) == "numeric")     # Target feature: numeric
cat("Numerical Features(num):", names(Numeric))

```

However, the documentation on the data [@de2011ames] says that the data-set consist of 20 continuous features that refer to area dimensions, 14 discrete features that quantify the number of items in the house, 23 nominal categorical features that refer to types of dwellings, materials and conditions, and 23 ordinal categorical features that rate various property related items. After correcting the data types according to the documentstion, it should look as  shown in Table 3.7. 

```{r, eval=TRUE, echo=FALSE}

#How the data should be :
Cont_Features_20<- c("LotFrontage", "MasVnrArea", "BsmtFinSF1", "BsmtFinSF2", "BsmtUnfSF", "TotalBsmtSF", "GarageArea", "X1stFlrSF", "X2ndFlrSF", "LowQualFinSF", "GrLivArea", "X3SsnPorch", "PoolArea", "WoodDeckSF", "SalePrice" ,"EnclosedPorch", "ScreenPorch", "LotArea",  "MiscVal", "OpenPorchSF", " ", " ", " ")

Disc_Features_14<- c("BsmtFullBath","BsmtHalfBath","FullBath", "HalfBath", "BedroomAbvGr", "KitchenAbvGr", "TotRmsAbvGrd", "Fireplaces", "GarageCars","GarageYrBlt","YearBuilt","YearRemodAdd", "MoSold", "YrSold", " "," ", " ", " ",  " ", " ", " ", "", "")

Nom_Categorical_23<- c("Neighborhood", "SaleCondition", "HouseStyle", "Street", "Alley", "CentralAir", "LandContour", "Condition1", "Condition2", "BldgType", "RoofStyle", "RoofMatl", "Exterior1st", "Exterior2nd", "Foundation", "Heating", "GarageType", "MiscFeature", "SaleType", "MSSubClass","MSZoning", "MasVnrType", "LotConfig")
  
Ord_Categorical_23<- c("Utilities", "LandSlope", "ExterQual", "ExterCond","BsmtQual","BsmtCond","BsmtExposure","BsmtFinType1","BsmtFinType2",
"HeatingQC","Electrical","KitchenQual","Functional","FireplaceQu", "GarageFinish","GarageQual","GarageCond","PavedDrive","PoolQC","Fence","OverallQual","OverallCond", "LotShape" )

data_types<-data.frame(Cont_Features_20,Disc_Features_14,Nom_Categorical_23,Ord_Categorical_23)
kable(data_types,caption = "How the Data Should Look")

```
**The first step is to convert the numeric features to the appropriate data type.**
```{r,eval=TRUE,collapse = TRUE}
to_num <-c( "LotFrontage", "MasVnrArea", "BsmtFinSF1", "BsmtFinSF2", "BsmtUnfSF", "TotalBsmtSF", "GarageArea","X1stFlrSF", "X2ndFlrSF", "LowQualFinSF", "GrLivArea", "X3SsnPorch", "PoolArea", "WoodDeckSF", "SalePrice" ,"EnclosedPorch", "ScreenPorch", "LotArea",  "MiscVal", "OpenPorchSF")
data_full[,to_num] <- lapply(data_full[,to_num], as.numeric)
to_int<-c("BsmtFullBath","BsmtHalfBath","FullBath", "HalfBath", "BedroomAbvGr", "KitchenAbvGr", "TotRmsAbvGrd", "Fireplaces", "GarageCars", "MoSold", "YrSold","YearRemodAdd","GarageYrBlt","YearBuilt")
data_full[,to_int] <- lapply(data_full[,to_int], as.integer)
```
**The second step, is to convert missclassified numeric features to categorical.**
```{r, eval=TRUE, collapse = TRUE}
nom_to_cat <-c("MasVnrType","MSSubClass", "Neighborhood","CentralAir", "SaleCondition", "HouseStyle", "Street", "Alley", "LandContour", "Condition1", "Condition2", "BldgType", "RoofStyle", "RoofMatl", "Exterior1st", "Exterior2nd", "Foundation", "BsmtExposure", "Heating", "GarageType", "PavedDrive", "MiscFeature", "SaleType")
data_full[,nom_to_cat] <- lapply(data_full[,nom_to_cat], factor)
```
**The third step is to add levels to ordinal categorical features.**
```{r, eval=TRUE, collapse = TRUE}
#3. Add order to levels for ordinal variables 

data_full$Utilities<-factor(data_full$Utilities,levels=c("ELO","NoSeWa","NoSewr","AllPub"), ordered=TRUE)

data_full$ExterQual<-factor(data_full$ExterQual, levels=c("Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$ExterCond<-factor(data_full$ExterCond, levels=c("Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$FireplaceQu<-factor(data_full$FireplaceQu, levels=c("None","Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$Functional<-factor(data_full$Functional,levels=c("Sal", "Sev", "Maj2", "Maj1","Mod","Min2","Min1","Typ"), ordered=TRUE)

data_full$PoolQC<-factor(data_full$PoolQC,levels=c("None","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$BsmtCond<-factor(data_full$BsmtCond, levels=c("None","Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$BsmtQual<-factor(data_full$BsmtQual,levels=c("None","Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$BsmtExposure<-factor(data_full$BsmtExposure, levels=c("None", "No", "Mn", "Av", "Gd"), ordered=TRUE)

data_full$BsmtFinType1<-factor(data_full$BsmtFinType1, levels=c("None","Unf","LwQ","Rec","BLQ","ALQ","GLQ"), ordered=TRUE)

data_full$BsmtFinType2<-factor(data_full$BsmtFinType2, levels=c("None","Unf","LwQ","Rec","BLQ","ALQ","GLQ"), ordered=TRUE)

data_full$HeatingQC<-factor(data_full$HeatingQC, levels=c("Po", "Fa", "TA", "Gd", "Ex"), ordered=TRUE)

data_full$KitchenQual<-factor(data_full$KitchenQual, levels=c("Po", "Fa", "TA", "Gd", "Ex"), ordered=TRUE)

data_full$GarageQual<-factor(data_full$GarageQual, levels=c("None","Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$GarageCond<-factor(data_full$GarageCond, levels=c("None","Po","Fa","TA","Gd","Ex"), ordered=TRUE)

data_full$Electrical<-factor(data_full$Electrical, levels=c("Mix","FuseP","FuseF","FuseA","SBrkr"), ordered=TRUE)

data_full$GarageFinish<-factor(data_full$GarageFinish, levels=c("None","Unf","RFn","Fin"), ordered=TRUE)

data_full$PavedDrive<-factor(data_full$PavedDrive, levels=c("N","P","Y"), ordered=TRUE)

data_full$Fence<-factor(data_full$Fence, levels=c("None","MnWw","GdWo", "MnPrv", "GdPrv"), ordered=TRUE)

data_full$OverallQual<-factor(data_full$OverallQual, levels=c("1", "2","3","4", "5", "6", "7", "8", "9", "10"), ordered=TRUE)

data_full$OverallCond<-factor(data_full$OverallCond, levels=c("1", "2","3","4", "5", "6", "7", "8", "9", "10"), ordered=TRUE)

data_full$GarageType<-factor(data_full$GarageType, levels=c("None","2Types","Attchd", "Basment","BuiltIn","CarPort","Detchd" ), ordered=FALSE)

data_full$LandSlope<-factor(data_full$LandSlope, levels=c("Gtl","Mod","Sev" ), ordered=TRUE)

data_full$LotShape<-factor(data_full$LotShape, levels=c("IR1","IR2", "IR3", "Reg" ), ordered=TRUE)

```
##### Confirm all features are now approrpiately classified by their data type.

```{r data_type_corrected, echo=FALSE, eval=TRUE, collapse = TRUE,comment="***" }

integer_var <- which(lapply(data_full,class) == "integer")     # Integer features
cat(length(integer_var),"Discrete Features:", names(integer_var))

numeric_var <- which(lapply(data_full, class) == "numeric")     # Target feature: numeric
cat(length(numeric_var),"Continous Feature:", names(numeric_var ))

factor_var <- which(lapply(data_full, class) == "factor")    # Categorical features
cat(length(factor_var), "Nominal Categorical Features:", names(factor_var))

factor_var2 <- which(sapply(data_full, is.ordered))  # Categorical features
cat(length(factor_var2),"Ordered Categorical Features:", names(factor_var2))

char_var <- which(lapply(data_full, class) == "character") # Zero character types 

```

## Feature Transformation

With clean data in the correct form, it is time to start the Exploratory Data Analysis (EDA). By visualizing the data we hope to uncover interesting patterns and relationships which we could use to enhance the predictive value of the features via transformation or new feature creation. 

 
### Visualizing the Distribution and Spread of Target Feature : SalesPrice 

The histogram of the "SalesPrice" feature in Figure 4. shows a few very large values on the right, making the distribution of the data right skewed. Since the goal is to predict the continuous numerical variable "SalesPrice" with regression models, it might be useful to transform it, since one of the assumptions of regression  analysis is that the error between the observed and expected values (the residuals) should be normally distributed, and violations of this assumption often stem from a skewed response variable. We can make the distribution more normal by taking the natural logarithm, since in a right-skewed distribution where there are a few very large values, the log transformation helps bring these values into the center. After applying the log transformation, as seen in  Figure 5., the distribution looks more symmetrical.
```{r SalePriceHistogram, echo=TRUE, collapse = TRUE}
#Create a copy of the whole dataset to work from here on out
data_2<-data_full  
#Create a set that includes only the training examples
data_2_train<-data_2[1:1460,]
par(mfrow=c(1,1))
ggplot(data_2_train, aes(SalePrice)) + ggtitle("Target Feature: SalePrice")+
geom_histogram(fill="#053061", alpha=.5, color="#F5F5F5" , bins = 30)+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE))+
        geom_vline(aes(xintercept = mean(SalePrice), 
                   color = "mean"), linetype = "dashed", size = .7) +
        geom_vline(aes(xintercept = median(SalePrice), 
                   color = "median"), linetype = "dashed", size = .7) +
        scale_color_manual(name = "Central Tendency", values = c(mean = "#67001F", median = "#01665E"))
#Log transform the Target Feature
data_2_log<-data_2_train
data_2_log$SalePrice<-log(data_2_log$SalePrice)

ggplot(data_2_log, aes((SalePrice))) + ggtitle("SalePrice with Log Transformation")+
geom_histogram(fill="#053061", alpha=.5, color="#F5F5F5", bins = 30)+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) +
        geom_vline(aes(xintercept = mean(SalePrice), 
                   color = "mean"), linetype = "dashed", size = .7) +
        geom_vline(aes(xintercept = median(SalePrice), 
                   color = "median"), linetype = "dashed", size = .7) +
        scale_color_manual(name = "Central Tendency", values = c(mean = "#67001F", median = "#01665E"))

```


#### Features Highly Correlated with Sales Price 

Given the large number of features in the data-set, we will focus our data engineering efforts on those features which are highly correlated with our variable of interest. 

For continuous features, the correlation plot shows Garage Area, Great Living Room Area, First Floor SF, Total Basement SF, and Masonry Veneer Area have a strong positive correlation with Sale Price. 


```{r cont_Corr, collapse = TRUE}
ggcorr(data_2_train[,numeric_var], label = TRUE, label_size = 2.9, hjust = 1, layout.exp = 2)+ ggplot2::labs(title = "Continous Features and Sale Price Correlation Plot")

```
^[The correlation plot was produced with the ggcorr() function from package 'GGally' [@GGally].]

From the histograms we can see that many of the continuous independent variables are right skewed, similar to the response, so it might be a good idea to normalize these prior to modeling, since many algorithms perform better with normalized data because it improves the numerical stability of the model and reduces training time (Zhang_2019). 


```{r, ContinousFeatures2, collapse = TRUE, echo=FALSE}
#Garage Area
gar_a<-ggplot(data_2_train, aes(x=GarageArea)) + ggtitle("Garage Area")+ geom_histogram(aes(y=..density..), bins = 30, colour="#F5F5F5", fill="#053061", alpha=.5) + geom_density(adjust = 5, colour= "#053061") +  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE))+
        geom_vline(aes(xintercept = mean(GarageArea), 
                   color = "mean"), linetype = "dashed", size = .7) +
        geom_vline(aes(xintercept = median(GarageArea), 
                   color = "median"), linetype = "dashed", size = .7) +
        scale_color_manual(name = "Central Tendency", values = c(mean = "#B2182B" , median = "#01665E"))

# Living Area
liv_a<-ggplot(data_2_train, aes(x=GrLivArea)) + ggtitle("Living Room Area")+ geom_histogram(aes(y=..density..), bins = 30, colour="#F5F5F5", fill="#053061", alpha=.5) + geom_density(adjust = 5, colour= "#053061") 

#First Floor SF
first_f<-ggplot(data_2_train, aes(x=X1stFlrSF)) + ggtitle("First Floor SF")+ geom_histogram(aes(y=..density..), bins = 30, colour="#F5F5F5", fill="#053061", alpha=.5) + geom_density(adjust = 5, colour= "#053061") 

#Total Basement SF

tot_bas<-ggplot(data_2_train, aes(x=TotalBsmtSF)) + ggtitle("Total Basement SF")+ geom_histogram(aes(y=..density..), bins = 30, colour="#F5F5F5", fill="#053061", alpha=.5) + geom_density(adjust = 5, colour= "#053061") 

#and Masonry Veneer Area

mans_a<-ggplot(data_2_train, aes(x=MasVnrArea)) + ggtitle("MasVnrArea")+ geom_histogram(aes(y=..density..), bins = 30, colour="#F5F5F5", fill="#053061", alpha=.5) + geom_density( colour= "#053061")

grid.arrange(mans_a, liv_a, first_f, tot_bas, 
             ncol = 2, nrow = 2)

```
^[The histograms were produced with ggplot() function from package 'ggplot2' [@ggplot2].]

The correlation plot for the discrete features in Figure 8. shows that YearBuilt, YearRemodAdd, GarageYearBuilt, GarageCars, FullBath, FirePlaces, and TotRmsAbvGrd are strongly and positively correlated with SalePrice: 
```{r, discCorr,echo=FALSE, collapse = TRUE}

ggcorr(data_2_train[,c(integer_var[-1], 81)], label = TRUE, label_size = 2.9, hjust = 1, layout.exp = 2)+ ggplot2::labs(title = "Discrete Features and Sale Price Correlation Plot")

```
^[The correlation plot was produced with the ggcorr() function from package 'GGally' [@GGally].]

#### Attribute Construction from Year Features 

The barplot of Year Built shows a clear distinction between the number of houses built and sold in the 2000s vs the number built and sold in before that time. The distribution is left skewed, as we can see that the mean of the distribution is less than the median. Most importantly, since there are so many unique years, it might be helpful to create a new “age” variable by subtracting the year built from 2010, which is the upper limit of
year sold in the data-set. And we can further refine this new age variable later on by binning age groups
into buckets from old to new, thereby reducing the number of levels.

```{r, YearBuilt, collapse = TRUE, echo=FALSE}
ggplot(data_2_train, aes(YearBuilt))+
  geom_bar(fill="#053061", color="#F5F5F5")+
        geom_vline(aes(xintercept = mean(YearBuilt), 
                   color = "mean"), linetype = "dashed", size = .7) +
        geom_vline(aes(xintercept = median(YearBuilt), 
                   color = "median"), linetype = "dashed", size = .7) +
        scale_color_manual(name = "Statistics", values = c(mean = "red", median = "darkgreen")) +
        ggtitle("Bar Plot of Year Built") +
        xlab("Year Built") + ylab("Number of Homes")

```
^[The bar-plot was produced with ggplot() function from package 'ggplot2' [@ggplot2].]

Curiously, in the scatter plot shown in Figure 10., where the dots have been colored in light blue by the YearRemodAdd feature,  newer houses are displaying as having been remodeled, apparently if the house has not been remodeled the year remodeled defaults to the year built. To improve the value of this metric we can create a new binary feature for Remodeled: Yes or No. Also, it might be useful to create a feature to show how recently the house was remodeled, and bin these into categories from most recently remodeled. 

```{r YearRemodeled, collapse = TRUE}
ggplot(data_2_train, aes(x=YearBuilt, y=SalePrice, color=(YearRemodAdd))) + geom_jitter(height = 1) + labs(title="Year Built vs Sale Price Colored by Year Remodeled ", subtitle="Training Data")
```

^[The scatter plot was produced with ggplot() function from package 'ggplot2' [@ggplot2].]

In order to make the year features more meaningful, we created new attributes as indicated in the table below. In addition to the building age and  time since remodeled features, we added a feature to indicate if the home is a new build, since it is likely that being a new home will have an impact on the sale's price. We also added a feature for the year the house was sold, since macro economic events, like the economic depression in 2008, could also impact the sale price. Later, we simplify some of these newly created features by binning them into categories with fewer levels. Since all of the categorical features will have to be converted into numeric via dummy coding for modeling, having fewer levels will help with model performance and dimensionality reduction.

**Attribute Construction from Year Features**

| New Feature    | Method                                                    |
| :------------- | :-------------------------------------------------------- |
| BldgAge        | 2010 - YearBuilt                                          |
| NewBuild       | If YearBuilt or YearBuilt + 1 =  YrSold then NewBuild = 1 |
| Remod          | If YearBuilt = YearRemodAdd then Remod = 0                |
| TimeSinceRemod | 2010 - YearRemodAdd                                       |
| LastSold       | 2010 - YrSold                                             |



\begin{table}[h]
  \begin{center}
  \begin{tabular}{ |c|c|c| } 
    \hline
    \multicolumn{2}{|c|}{Attribute Construction from Year Features} \\
    \hline
    New Feature & Method \\
    \hline\hline
    BldgAge  &  2010 $-$ YearBuilt \\
    NewBuild & If YearBuilt or YearBuilt $+$ 1 $=$  YrSold then NewBuild =1  \\
    Remod & If YearBuilt $=$ YearRemodAdd then Remod=0 \\ 
    TimeSinceRemod & 2010 $-$ YearRemodAdd  \\
    LastSold & 2010 $-$ YrSold \\
    \hline
  \end{tabular}
  \end{center}
\caption{Year Features}
\end{table}


```{r, NewYearFeatures, collapse = TRUE}

# Remodeled
data_2['Remod'] <- ifelse(data_2$YearBuilt==data_2$YearRemodAdd, 0, 1) #0=No Remodeling, 1=Remodeling
#New Build 
data_2['NewBuild'] <-  ifelse(data_2$YearBuilt == data_2$YrSold|data_2$YearBuilt+1 == data_2$YrSold, 1,0) 
# Age of property based on last year of dataset
data_2['BldgAge']<- max(data_2$YearBuilt) - data_2$YearBuilt
head(data_2[,c("YearBuilt","YrSold", "BldgAge","YearRemodAdd", "Remod", "NewBuild")])

```

#### Reducing Levels by Grouping Unrepresented Categories
Another way that we can improve the data for mining is to reduce the levels in categorical variables by merging under-represented categories. For example, in the case of the fireplaces, we can see that there are only a few houses that have 3 fireplaces, and the scatter plot shows that the relationship between three fireplaces and price is not much different than between two fireplaces and price, so we can merge these into one level. 

Similarly, the scatter plot in Figure 13. of the "GarageCars" variable shows that few houses have more than three garages, so we could simplify the levels in this feature by keeping only three levels: 1,2, and 3+. 


```{r, GarageFeature, collapse = TRUE}

fireplaces_bar<-ggplot(data_2_train)+aes(Fireplaces)+ labs(title="Fireplaces vs Sale Price", subtitle="Training Data")+
geom_bar(fill="#92C5DE", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

fireplaces_scatter<-ggplot(data_2_train, aes(x=Fireplaces, y=SalePrice, color=(Fireplaces))) + geom_jitter(height = 1) + labs(title="Fireplaces vs Sale Price", subtitle="Training Data")

 garagecars_bar<-ggplot(data_2_train)+aes(GarageCars)+ labs(title="Garage vs Sale Price", subtitle="Training Data")+
geom_bar(fill="#92C5DE", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

garagecars_scatter<-ggplot(data_2_train, aes(x=GarageCars, y=SalePrice, color=(GarageCars))) + geom_jitter(height = 1) + labs(title="Garage vs Sale Price", subtitle="Training Data")


grid.arrange(fireplaces_bar,fireplaces_scatter, garagecars_bar, garagecars_scatter,
             ncol = 2, nrow = 2)


```
^[The plots were produced with ggplot() function from package 'ggplot2' [@ggplot2].]

We applied this treatment to the features listed in Table below. This will make these features more robust for modeling. 

**Reducing Levels by Grouping**

| New Feature | Method                             |
| :-----------| :--------------------------------- |
| GarageCars  | Merge 4 and 3                      |
| Fireplaces  | Merge 3 and  2                     |
| Electrical  | Merge FuseF and FuseA              |
| Function    | Merge Maj1 and Mod, Min1 and Min2  |
                                

\begin{table}
  \begin{center}
  \begin{tabular}{ |c|c|c| } 
  \hline
  \multicolumn{2}{|c|}{Reducing Levels by Grouping} \\
  \hline
  New Feature & Method \\
  \hline\hline
  GarageCars & Merge 4 and 3 \\
  Fireplaces & Merge 3 and  2 \\ 
  Electrical &  Merge FuseF and FuseA\\
  Function & Merge Maj1 and Mod, Min1 and Min2 \\
  \hline
  \end{tabular}
  \end{center}
\caption{Grouping}
\end{table}


**GroupingGarageCars Features**

```{r GrouppingGaragecars, collapse = TRUE}
#Recode sparcely populated categories to reduce the number of levels 

#Garage Cars
data_2$GarageCars[data_2$GarageCars==4]<-3
data_2$GarageCars[data_2$GarageCars==5]<-3

ggplot(data_2[1:1460,])+aes(GarageCars)+ labs(title="GarageCars", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

ggplot(data_2[1:1460,], aes(x=GarageCars, y=SalePrice, color=(GarageCars))) + geom_jitter(height = 1) + labs(title="GarageCars vs Sale Price", subtitle="Training Data")

```

**Grouping Fireplace Features**
```{r GrouppingFireplaces, collapse = TRUE}
#FirePlaces 
data_2$Fireplaces[data_2$Fireplaces==3]<-2
data_2$Fireplaces[data_2$Fireplaces==4]<-2

ggplot(data_2[1:1460,])+aes(Fireplaces)+ labs(title="Number of Fireplaces", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

ggplot(data_2[1:1460,], aes(x=as.factor(Fireplaces), y=SalePrice, color=Fireplaces)) + geom_jitter(height = 1) + labs(title="Fireplaces vs Sale Price", subtitle="Training Data") 

```

**Grouping Electrical Features**

```{r Groupping#Electrical , collapse = TRUE}
data_full$Electrical<-factor(data_full$Electrical, levels=c("Mix","FuseP","FuseF","FuseA","SBrkr"), ordered=TRUE)
data_2$Electrical<-recode_factor(data_2$Electrical, "Mix"=1, "FuseP"=2, "FuseF"=3, "FuseA"=3, "SBrkr"=4)
ggplot(data_2[1:1460,], aes(x=Electrical, y=SalePrice, color=Electrical)) + geom_jitter(height = 1) + labs(title="Electrical vs Sale Price", subtitle="Training Data")

```

**Grouping Functional Features** 

```{r GrouppingFunctional, collapse = TRUE}
    
data_2$Functional<-recode_factor(data_2$Functional,  "Sal"=1 , "Sev"=1 , "Maj2"=2 ,"Maj1"=3 ,"Mod"=3,  "Min2"=4 ,"Min1"=4 ,"Typ"=5)
sum(is.na(levels(data_2$Functional)))

ggplot(data_2[1:1460,], aes(x=Functional, y=SalePrice, color=(Functional))) + geom_jitter(height = 1) + labs(title="Functional vs Sale Price", subtitle="Training Data")


```

#### Attribute Construction by combining some features to make new features. 
There are four different columns related to the number of bathrooms in different areas of the home.  Individually these features may not carry as mush weight as combined into a single feature for total number baths. 

```{r BathroomFeatures,  collapse = TRUE}

bsmtFB<-ggplot(data_2_train)+aes(BsmtFullBath)+ labs(title="Basement Full Bath", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

BsmtHB<-ggplot(data_2_train)+aes(BsmtHalfBath)+ labs(title="Basement Half Bath", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

hFB<-ggplot(data_2_train)+aes(FullBath)+ labs(title="Number of Full Baths in the Home", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE))

hhB<-ggplot(data_2_train)+aes(HalfBath)+ labs(title="Number of Half Baths in the Home", subtitle="Training Data")+
geom_bar(fill="#053061", color="#F5F5F5")+  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 

grid.arrange(bsmtFB, BsmtHB, hFB, hhB, 
             ncol = 2, nrow = 2)

```

^[The plots were produced with ggplot() function from package 'ggplot2' [@ggplot2].]

Some homes are showing as having zero baths, this might be a mistake, so it will be usefull to look closer are these examples. The nine rows which had zero full bath, did have other bathrooms in either the basement, or half bath, it is ok to leave them for now, since it will be better to combine all baths together and have one feature for the total baths. 

**Close look at fullbath = 0 **

```{r, BathroomZero,  collapse = TRUE}

missing_bath <- filter(data_2_train, FullBath ==0)
bath_features <- names(data_2_train)[sapply(names(data_full), function(x) str_detect(x, "Bath"))]

data_2_train[c(54, 189, 376, 598, 635, 917, 1164,1214, 1271), c("BsmtFullBath","BsmtHalfBath","FullBath","HalfBath")]

```

We will do the same to the basement finished squared feet and the porch features. Furthermore, we created a new feature for total area by adding the living area square feet and the total basement square feet.  The goal here is to create new features that might have a stronger predictive power than the individual original features. Prior to modeling, we will be removing all of the features rendered redundant by the feature creation process. The table below summarizes the features we constructed by adding the same type of feature together to form a total.

**New Feature Construction by Combining Existing Features**

| New Feature   | Method                                                  |
| :------------ | :------------------------------------------------------ |
| BSmtFinSFComb | BsmtFinSF1 + BsmtFinSF2                                 |
| TotalArea     | GrLivArea + TotalBsmtSF                                 |
| TotalBaths    | FullBath + HalfBath + BsmtFullBath + BsmtHalfBath       |
| TotalPorchSF  | OpenPorchSF + EnclosedPorch + X3SsnPorch + ScreenPorch  |
            

\begin{table}[h]
  \begin{center}
  \begin{tabular}{|c|c||c|} 
  \hline
  \multicolumn{2}{|c|}{New Feature Construction by Combining Existing Features} \\
  \hline
  New Feature & Method \\
  \hline\hline
  BSmtFinSFComb & Adding BsmtFinSF1 $+$ BsmtFinSF2 finished floors \\
  TotalArea  & Adding GrLivArea $+$ TotalBsmtSF \\
  TotalBaths & FullBath $+$ HalfBath $+$ BsmtFullBath $+$ BsmtHalfBath \\
  TotalPorchSF & Adding OpenPorchSF $+$ EnclosedPorch $+$ X3SsnPorch $+$ ScreenPorch \\ 
  \hline
  \end{tabular}
  \end{center}
\caption{Combining}
\end{table}


```{r CombineFeatures, collapse = TRUE}
data_2 <- mutate(data_2, BSmtFinSFComb = BsmtFinSF1 + BsmtFinSF2,
                   TotalArea= GrLivArea + TotalBsmtSF,
                   TotalBaths = BsmtFullBath + (0.5*BsmtHalfBath) + FullBath+ (0.5*HalfBath),
                   PorchSF = OpenPorchSF + EnclosedPorch + X3SsnPorch + ScreenPorch)

#To view cobined fatures
#data_2[,c("X1stFlrSF","X2ndFlrSF", "BSmtFinSFComb", "TotalBsmtSF", "GrLivArea", "TotalArea")]

```

#### Attribute Construction by Binning
Some of the categorical variables have a large number of levels. For example, "Neighborhood",  one of the features most highly correlated with sale price, has 25 different levels. 

```{r Neighborhood_plots}

ggplot(data_2_train)+ 
  geom_boxplot(mapping=aes(x=reorder(Neighborhood,SalePrice, FUN=median), y=SalePrice), fill = "#053061", color="#053061", alpha=.3,)+ggtitle("Neighborhood vs Sale Price")+theme(axis.text.x = element_text(angle = 45,hjust = 1), # use inclined text on the x-axis
              plot.title = element_text(hjust = 0.5, vjust = 3, size = 14, face = "bold"), 
              plot.subtitle = element_text(hjust = 0.5, vjust = 3, size = 12, face = "italic"))
neig_tabl<-table(data_2$Neighborhood)
kable(sort(neig_tabl,decreasing=T))

```

#### Data Transformation: Descretization and Binning-Neighboorhood
Concept hirearchy discretization and binning is a way to reduce the the number of distinc values per attribute, we will group the neighborhoods together to create fewer categories.  We can reference the  SalePrice to find similar neighboorhoods. Alternatively, in order to avoid data leakage we could use the “Overall Quality” feature, which shows a similar relationship with the neighborhoods,to create 5 bins for neighborhood categories ranging from low (1) to high (5).

Most expensive neighborhoods: ‘StoneBr’, ‘NridgHt’ and ‘NoRidge’.

Least expensive neighborhoods: ‘MeadowV’, ‘BrDale’, and ‘IDOTRR’.

```{r NighborhoodsSummary, collapse = TRUE}
# Summary statistics
ames_train<-as.data.frame(data_2[1:1460,])
log_price<-log(ames_train$SalePrice)
neigh_brk<-setDT(ames_train)[ , list(mean_1 = mean(SalePrice), median_1=median(SalePrice), st_dev = sd(SalePrice),  sum_gr = sum(SalePrice), range=range(SalePrice)[2] - range(SalePrice)[1]) , by = .(Neighborhood)]
```


```{r NeighSummPlot, collapse = TRUE}
# Sort by median and show top, bottom
top_10<-kable(neigh_brk %>%
        arrange(desc(median_1)) %>%
        head(10))

bottom_10<-kable(neigh_brk %>%
        arrange(desc(median_1)) %>%
        tail(10))

```


```{r NeighComm, collapse = TRUE, echo=FALSE}
cat("\n")
cat("Top 10 Neighborhoods by Median Sales Price:")
top_10
cat("\n")

cat("\n")
cat("Bottom 10 Neighborhoods by Median Sales Price:")
bottom_10
cat("\n")

cat("\n")
cat("All Neighborhoods by Median Sales Price:")
cat("\n")

neigh_brk %>%
        arrange(desc(median_1))
```


```{r NeighHet, collapse = TRUE, echo=FALSE}
cat("The most heterogeneous neighborhoods: 'NoRidge', ‘StoneBr’, ‘NridgHt’, and 'Veekner'.")
neigh_brk %>%
        arrange(desc(st_dev)) %>%
        head(6)
```

```{r NighborhoodRidges, collapse = TRUE}
data_2[1:1460,] %>%
        ggplot(aes(x = SalePrice, y = Neighborhood, fill = Neighborhood)) +
        geom_density_ridges() +
        theme_ridges() +
        theme(legend.position = "none") +
        xlab("Sale Price") +
        theme(axis.title.y = element_blank())
```

**Grouping Neighborhoods**

```{r Nighborhood Grouping,  collapse = TRUE}

data_2$Neigh_Cat<-recode_factor(data_2$Neighborhood, 'MeadowV' = 1, 'IDOTRR' = 1, 'BrDale' = 1, 'BrkSide' = 2, 'Edwards' = 2,'OldTown' = 2,'Sawyer' = 2, 'Blueste' = 2, 'SWISU' = 2,  'NPkVill' = 2, 'NAmes' = 2, 'Mitchel' = 2,'SawyerW' = 3,'NWAmes' = 3,  'Gilbert' = 3, 'Blmngtn' = 3, 'CollgCr' = 3, 'Crawfor' = 4, 'ClearCr' = 4,'Somerst' = 4, 'Veenker' = 4, 'Timber' = 4, 'StoneBr' = 5, 'NoRidge' = 5, 'NridgHt' = 5)

data_2$Neigh_Cat <- as.numeric(data_2$Neigh_Cat)
#data_2[, c('Neigh_Cat', 'Neighborhood')]

#plot quality vs neighborhood 
ggplot(data_2[1:1460,], aes(x=Neigh_Cat, y=OverallQual, color=(Neigh_Cat))) + geom_point(alpha=0.2) + geom_jitter(height = 1.5) + labs(title="Quality and Neighborhood Category") +theme(axis.text.x = element_text(angle = 45,hjust = 1))

ggplot(data_2, aes(x=Neigh_Cat, y=OverallQual, color=(Neighborhood))) + geom_point() + geom_jitter(height = 1.5) + labs(title="Quality and Neighborhood Category") +theme(axis.text.x = element_text(angle = 45,hjust = 1))

ggplot(data_2[1:1460,], aes(x=Neigh_Cat, y=SalePrice, color=(Neighborhood))) + geom_point(alpha=0.2) + geom_jitter(height = 1.5) + labs(title="Sale Price and Neighborhood Category") +theme(axis.text.x = element_text(angle = 45,hjust = 1))

ggplot(data_2[1:1460,], aes(x=Neigh_Cat, fill=factor(Neigh_Cat))) + geom_bar() + scale_fill_brewer(palette = "Blues") + labs(title= "New Neighborhood Category") +theme(axis.text.x = element_text(angle = 45,hjust = 1) )

```
^[The plots were produced with ggplot() function from package 'ggplot2' [@ggplot2].]

In addition to the "Neighborhood" feature, we also used binning as a means to reduce the number of unique levels for the "Month" feature.  For the Month feature, we ploted the months, and put them into bins according to low, medium, or high season, based on the number of houses sold. The AgeCat and RemodelFromCat are new features based of other features we contructed previously from the year features. These features contain fewer levels, which is one of our goals here, to reduce the nuber of levels in categorical features, so that the prediction model is simpler, which as mentioned before reduces the error.  The table below shows the attributes constructed by binning.

**Attribute Construction by Binning**

| New Feature    | Method                                                    |
| :------------- | :-------------------------------------------------------- |
| AgeCat         | BldgAge into 4 age categories: Antique, Old, Mid, New     |
| RemodelFromCat | Max year in Data (2010) - YearRemodAdd into 4 categories  |
| SeasonSale     | MoSold into 3 categories (Low, Mid & High)                |
| NeighCat       | Neighborhood into 5 categories                            | 
            
\begin{table}
  \begin{center}
  \begin{tabular}{ |c|c|c| } 
  \hline
  \multicolumn{2}{|c|}{Attribute Construction by Binning} \\
  \hline
  New Feature & Method \\
  \hline\hline
  AgeCat & BldgAge into 4 age categories: Antique, Old, Mid, New \\
  RemodelFromCat & Max year in Data (2010) $-$ YearRemodAdd into 4 categories \\ 
  SeasonSale &  MoSold into 3 categories for Low, Mid and High season\\
  NeighCat & Neighborhood into 5 categories based on OveralQuality \\
  \hline
  \end{tabular}
  \end{center}
\caption{Binning}
\end{table}

**Bin the age of the house into 4 categories** 

```{r, BinningAge,collapse = TRUE}
min(data_2$BldgAge) 
max(data_2$BldgAge) 

head(data_2[,c("BldgAge","YearBuilt")])

age<-data_2$BldgAge
AgeCat<- case_when(age<= 9 ~ 'New',
                  between(age, 10, 40) ~ 'Mid',
                  between(age, 41, 70) ~ 'Old',
                  age >= 71 ~ 'Antique')

data_2$AgeCat<-as.numeric(factor(AgeCat, levels=c('Antique','Old', 'Mid', 'New'), ordered=TRUE))
head(data_2[,c("BldgAge", "AgeCat","YearBuilt")])

#Time when property sold  based on max year of dataset
data_2$LastSold <- 2010 - data_2$YrSold  #This might be interesting to look at from economic boom and bust effects
```

**Bin the year-since-remodeled of the house into 4 categories** 

```{r BinningRemodeled, collapse = TRUE}
##Years Since Remodeled 
data_2['YearRemodAdd2'] <- ifelse(data_2$YearBuilt == data_2$YearRemodAdd, 0, data_2$YearRemodAdd) #fix the year remodeled column
head(data_2[,c("YearBuilt","YearRemodAdd", 'YearRemodAdd2','Remod' )])
data_2['TimeSinceRemod'] <- ifelse(data_2$Remod == 1, 2010 - data_2$YearRemodAdd2,0) 
data_2$RemodelFromCat<- case_when(data_2$TimeSinceRemod <= 0 & data_2$NewBuild == 1 ~ 'New',
                  data_2$TimeSinceRemod <= 0 & data_2$YearRemodAdd2 == 0 ~ 'None',
                  between(data_2$TimeSinceRemod, 0, 5) ~ 'Recent',
                  between(data_2$TimeSinceRemod, 6, 10) ~ 'Mid',
                  between(data_2$TimeSinceRemod, 11, 20) ~ 'Old',
                  data_2$TimeSinceRemod >= 20 ~ 'Outdated')
data_2$RemodelFromCat<-factor(data_2$RemodelFromCat, levels=c('None','Outdated','Old', 'Mid','Recent', 'New'), ordered=TRUE)
ggplot(data_2, aes(x=RemodelFromCat)) +
  geom_bar(fill = 'blue') +
  geom_text(aes(label=..count..), stat='count', vjust = -.5) +
  theme_minimal() 
data_2$RemodelFromCat<-as.integer(data_2$RemodelFromCat)
head(data_2[,c("YrSold","YearBuilt","YearRemodAdd", "YearRemodAdd2","Remod", 'TimeSinceRemod', 'BldgAge', 'NewBuild')])
```

**Bin the sale month into seasons from low season to high season based on sales** 

```{r BinningSeason, collapse = TRUE}
ggplot(data_2, aes(x=MoSold)) +
  geom_bar(fill = 'gray') +
  geom_text(aes(label=..count..), stat='count', vjust = -.5) +
  theme_minimal() 
data_2$SeasonSale <- mapvalues(data_2$MoSold, from = c("1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12"), to = c("LowSeason", "LowSeason", "MidSeason",'MidSeason', 'HighSeason','HighSeason','HighSeason','MidSeason', 'MidSeason','MidSeason','LowSeason','LowSeason'))
head(data_2[,c("MoSold","SeasonSale")])
data_2$SeasonSale<-as.numeric(factor(data_2$SeasonSale,levels=c('LowSeason','MidSeason', 'HighSeason'),ordered=TRUE))
head(data_2[,c("MoSold","SeasonSale")])

```

**Update Categorical Features to have numerical value**

```{r catetonum, collapse = TRUE}
data_2$Utilities <-as.numeric(data_2$Utilities)
data_2$LandSlope<-as.numeric(data_2$LandSlope)
data_2$ExterQual <-as.numeric(data_2$ExterQual)
data_2$ExterCond <-as.numeric(data_2$ExterCond)
data_2$BsmtQual <-as.numeric(data_2$BsmtQual)
data_2$BsmtCond <-as.numeric(data_2$BsmtCond)
data_2$BsmtExposure <-as.numeric(data_2$BsmtExposure)
data_2$BsmtFinType1 <-as.numeric(data_2$BsmtFinType1)
data_2$BsmtFinType2 <-as.numeric(data_2$BsmtFinType2)
data_2$HeatingQC <-as.numeric(data_2$HeatingQC)
data_2$CentralAir <-as.numeric(data_2$CentralAir)
data_2$Electrical <-as.numeric(data_2$Electrical)
data_2$KitchenQual <-as.numeric(data_2$KitchenQual)
data_2$Functional <-as.numeric(data_2$Functional)
data_2$FireplaceQu <-as.numeric(data_2$FireplaceQu)
data_2$GarageFinish <-as.numeric(data_2$GarageFinish)
data_2$GarageQual <-as.numeric(data_2$GarageQual)
data_2$GarageCond <-as.numeric(data_2$GarageCond)
data_2$PavedDrive <-as.numeric(data_2$PavedDrive)
data_2$PoolQC <-as.numeric(data_2$PoolQC)
data_2$Fence <-as.numeric(data_2$Fence)
data_2$OverallQual<-as.numeric(data_2$OverallQual)
data_2$OverallCond <-as.numeric(data_2$OverallCond)

```

#### New Variables by Interactions
The last type of feature engineering we attempted is attribute construction from interactions by multiplying certain features listed in the table below. 

**New Variables from Interactions**

| New Feature    | Method                     |
| :--------------| :------------------------- |
| OverallScore   | GarageQual * GarageCond    |
| GarageScore    | OverallQual * OverallCond  |
| ExterScore     | ExterQual * ExterCond      |
| KitchenScore   | KitchenAbvGr * KitchenQual |
| GarageGrade    | GarageArea * GarageQual    |           



```{r Interactions, collapse = TRUE}
data_2 <- mutate(data_2,GarageScore = GarageQual * GarageCond,
                   OverallScore= OverallQual * OverallCond,
                   ExterScore = ExterQual * ExterCond,
                   KitchenScore = KitchenAbvGr * KitchenQual,
                   GarageGrade = GarageArea * GarageQual)
```

## Dimensionality and Numerosity Reduction
The last major step left in pre-processing is dimensionality and numerosity reduction through the following steps:

* 1. Delete Redundant Features Due to Data Engineering
* 2. Delete Outliers
* 3. Delete Features with Near Zero Variability

```{r redundant}
data_2 <- subset(data_2, select = -c(BsmtFinSF1,BsmtFinSF2,BsmtFullBath,GarageArea,GarageQual,
                                         BsmtHalfBath,FullBath,HalfBath,X1stFlrSF, X2ndFlrSF,
                                         OpenPorchSF,EnclosedPorch,X3SsnPorch,ScreenPorch,
                                         GarageQual,GarageCond,ExterCond,KitchenAbvGr,KitchenQual,
                                         BsmtFinType1,BsmtFinType2,BsmtCond,BsmtQual,
                                         Neighborhood,YearBuilt,YrSold, BldgAge,YearRemodAdd, MoSold, GarageYrBlt,YearRemodAdd2, Remod, TimeSinceRemod,PoolQC))
```

### Outliers 
Accordying to the data collector: 
“There are 5 observations that an instructor may wish to remove from the data set before giving it to students. Three of them are true outliers (Partial Sales that likely don’t represent actual market values) and two of them are simply unusual sales (very large houses priced relatively appropriately). I would recommend removing any houses with more than 4000 square feet from the data set (which eliminates these 5 unusual observations)”

As we can see from the scatterplot 4 of the outlier are in the training set, which means that one is in the test set, because we are using the test set to submit to Kaggle.com for evalueation, we can not remove any rows from the test set (or it would be incomplete). Here, we will only remove the outliers indicated by the data collector in the training set only. Additional outlier detection could be useful in the feature. 
```{r OutliersID, collapse = TRUE}
highlight_df <- data_2_train %>% 
             filter(GrLivArea>=4000)

outlier_plot2<-ggplot(data_2_train, aes(x =GrLivArea , y = SalePrice, color="red")) + ggtitle("Outliers: Living Area vs SalesPrice")+
geom_point(color="#053061", alpha=.5)+ 
geom_point(data=highlight_df,
             aes(x=GrLivArea,y=SalePrice, color="red"))+
  theme(
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  axis.line = element_line(colour = "#053061"))+ scale_x_continuous(labels = function(x) format(x, scientific = FALSE)) 
outlier_plot2$labels$colour <- "Outliers"
outlier_plot2
```

**For now, we are only deleting the outliers in the training set.**

```{r outliersDel, collapse = TRUE}
highlight_df$SalePrice
data_2 <- data_2[-c(524, 692, 1183, 1299),]  
```

**View data shape after feature engineering**

```{r datashape, collapse = TRUE}
length(data_2)               
#sapply(data_2 , class)
names(data_2) 
```

**Export Pre-processed Data**

```{r cleandata, collapse = TRUE}
write.csv(data_2,"AmesDataClean.csv", row.names = FALSE)
```
**Clean Data**
The original data, the  "Ames Housing Data-set", provided at Kaggle.com, contains 2919 observations and 80 explanatory variables. However, from this point on we are starting with an already cleaned and pre-processed data-set based on previous work, which consists of 2919 observations and 67 variables.  After transforming the categorical variables to numerical with dummy coding, the columns increase to 167 columns. 

```{r cleanimport, echo=FALSE}
ames_clean<-read.csv("AmesDataClean.csv",stringsAsFactors = TRUE) #2915 obs. of  67 variables:
dim(ames_clean)
head(ames_clean[,1:15])
```

# Modeling


**Dummy Codding: One of the last pre-processing steps before modeling**
```{r dummycode}
#Convert Categorical Features to numeric with dummy codding using function dummyVars() from Caret package.
factor_var <- which(lapply(ames_clean, class) == "factor")
data_temp<-ames_clean
dummy<- dummyVars(" ~ MSSubClass + MSZoning +Street + Alley+ LotShape + LandContour+ LotConfig + Condition1 + Condition2 + BldgType + HouseStyle + RoofStyle + RoofMatl+ Heating + Exterior1st + Exterior2nd + MasVnrType + Foundation + GarageType + SaleType + SaleCondition + MiscFeature" , data = data_temp, fullRank = TRUE)
pred<- data.frame (predict(dummy, data_temp))
data_final<-cbind(ames_clean[,-factor_var], pred)

```
**Split pre-processed dataset into Train and Test sets for modeling***
```{r testtrainsplit, }
train_1<-data_final[1:1456,-1] 
train_x<-select(train_1, -SalePrice)
train_y<-train_1$SalePrice
test_1<-data_final[1457:2915,-1] 
test_1<-select(test_1, -SalePrice)
train_1$SalePrice<- log(train_1$SalePrice)
test_ID<-data_final[1457:2915,1]

```

## Multivariate Linear Regression Model
We start by fitting a multivariate linear regression model to illustrate the issue of multicollinearity present in our data. The model returns a warning: "Coefficients: (4 not defined because of singularities)". This is due to variables that have perfect multicollinearity, so we find those variables and remove them. 

```{r, fitlm1}
#Fit Linear Model
ml_model<- lm(SalePrice ~ ., train_1)
summary(ml_model)
```

```{r removecol}
#identify the linearly dependent variables
ld_vars <- attributes(alias(ml_model)$Complete)$dimnames[[1]]
#remove the linearly dependent variables variables
train_ld<-select(train_1, -all_of(ld_vars))
```

Then, we fit the model again, and take a closer look at the the Variance Inflation Factors (VIFs), which measure extent to which a predictor is correlated with the other predictor variables. As we can see in the output, we find high levels of multicollinearity with VIF values greater than 10 in many cases. Ideally, we would like to keep the VIF values below 5. 

```{r fitlm2, }
#run model again
ml_model_new <-lm(SalePrice ~ ., train_ld)
```

**Inspect the Variance Inflation Factors**
```{r VIF,}
vif(ml_model_new)
```
**Dealing with Multicollinearity**

The curse of dimensionality refers to the phenomenon that many types of data analysis become significantly harder as the dimensionality of the data increases (@tan2019). As more variables get added and the complexity of the patterns increase, the training of the model becomes more time consuming and the predictive power decreases. 

Multicollinearity for example, is a common problem in high‐dimensional data. Multicollinearity occurs when two or more predictor variables are highly correlated, and becomes a problem in many prediction settings. It is specially problematic in regression, since one of the assumptions of linear regression is the absence of multicollinearity and auto-correlation. As the number of features grow, regression  models tend to over-fit the training data, causing the sample error to increase. 

Some approaches for dimensionality reduction are linear algebra based techniques such as Principal Component Analysis(PCA), which finds new attributes (principal components) that are linear combinations of the original attributes orthogonal to each other, and which capture the maximum amount of variation in the data. Another approach is a filter method, where the features are selected prior to modeling for  their statistical value to the model, for example when features are selected for their correlation value with the predicted variable. Wrapper, methods are also popular, these include Forward and Backward Elimination. There are also some algorithms that have their built in functions for feature selection, like Lasso regression. 

This analysis deals with some of the issues that arise from high dimensionality by implementing regularized regression using the glmnet package [@glmnet]. We will also implement a wrapper method with a model called Boruta [@Boruta] to do feature selection  prior to fitting a gradient boosted model with the xgboost package[@xgboost].

## Model Selection 
One of the ways to deal with high multicollinearity is through regularization. Regularization is a regression technique, which limits, regulates or shrinks the estimated coefficient towards zero, which can reduce the variance and decrease out of sample error [@Boehmke]. This technique does not encourage learning of more complex models, and so it avoids the risk of over-fitting.

Three regularization methods that help with collinearity and over-fitting are:

* Lasso, penalizes the number of non-zero coefficients 
* Ridge, penalizes the absolute magnitude of the coefficients
* Elastic Net, a mixed method closely linked to lasso and ridge regression

In addition to the methods listed above, we will also try a different approach using feature subset selection with the Boruta package [@Boruta] prior to fitting a gradient boosted tree, with the XGBoost package.  

**Model Selection**

| Model   |  Model Type               | Tuning Parameters                 |
| :------ | :------------------------ | :-------------------------------- |
| Lasso   | Regularization            | lambda                            |
| Ridge   | Regularization            | lambda                            |
| Elastic | Regularization            | lambda                            |
| xgboost | Extreme Gradient Boosting | general & tree booster parameters |
                                       

```{r,echo=FALSE, message=FALSE, warning = FALSE}
#Use pre-process to finish the preprocessing steps 
#identify only the predictor variables
features <- setdiff(names(data_final), c("Sale_Price", "Id"))
#str(data_final) #183 variables
train_final<-data_final[1:1456,]
test_final<-data_final[1457:2915,]

# pre-process estimation based on training features
pre_process <- preProcess(
  x      = train_final[features],  #181 columns
  method = c("BoxCox", "center", "scale", "nzv"))

# apply to both training & test
train_x<- predict(pre_process, train_final[, features])
test_x<- predict(pre_process, test_final[, features])

```
## Regularized Regression Models 
The lasso model penalty is controlled by setting alpha=1, likewise the ridge penalty is controlled by alpha=0. The elastic-net penalty is controlled by alpha between 0 and 1. The tuning parameter lambda controls the overall strength of the penalty. 

### Identify the Optimal Lambda Parameter
To identify the optimal lambda value we used 10-fold cross-validation (CV) with cv.glmnet() from the glmnet package [@glmnet]. We selected the minimum lambda value as the metric to determine the optimal lambda value. 

The figures show the 10-fold CV MSE across all the log lamda values. The numbers across the top of the plot are the number of features in the model. The first dashed line represents the log lamda value with the minimum MSE, and the second is the largest log lamda value within one standard error of it. 

```{r dataprep,}
train_x2<-select(train_x, -SalePrice)
test_x2<-select(test_x, -SalePrice)
train_y<- log(train_final$SalePrice)
```
**Ridge Model***
```{r RidgeFit, fig.cap="Ridge Optimal Lamda"}
set.seed(123)
glm_cv_ridge <- cv.glmnet(as.matrix(train_x2), train_y, alpha = 0)
glm_cv_ridge
plot(glm_cv_ridge)
```

```{r RidgeFitLamda, echo=FALSE}
rfeatures<-nrow(extract.coef(glm_cv_ridge, lambda = "lambda.min")) -1
cat("The optimal lambda value selected for the ridge model is: ", glm_cv_ridge$lambda.min)
cat("And the ridge model retains all features, that is", rfeatures, "features in total.")
```

**Lasso Model***
```{r LassoFit, fig.cap="Lasso Optimal Lamda"}
set.seed(123)
glm_cv_lasso <- cv.glmnet(as.matrix(train_x2), train_y, alpha = 1)
glm_cv_lasso
plot(glm_cv_lasso)
```


```{r, echo=FALSE}
lfeatures<-nrow(extract.coef(glm_cv_lasso, lambda = "lambda.min")) -1
cat("The optimal lambda value selected for the lasso model is:", glm_cv_lasso$lambda.min) 
cat("And the number of features to be retained is:", lfeatures)
```

**Elastic Net Model ***
```{r, modelfitnet, fig.cap= "Elastic Net Optimal Lamda"}
set.seed(123)
glm_cv_net<- cv.glmnet(data.matrix(train_x2), train_y, alpha = 0.1)
glm_cv_net
plot(glm_cv_net)
```

```{r, echo=FALSE}
efeatures<-nrow(extract.coef(glm_cv_net, lambda = "lambda.min")) -1
cat("The optimal lambda value selected for the Elastic Net model is:", glm_cv_net$lambda.min) 
cat("And the number of features to be retained is:", efeatures)
```

## Training the Regularized Regression Models 
```{r modeltrain}
#Select lambda that reduces error
penalty_ridge <- glm_cv_ridge$lambda.min
penalty_lasso <- glm_cv_lasso$lambda.min
penalty_net <- glm_cv_net$lambda.min

set.seed(123)
glm_ridge_mod <- glmnet(x = as.matrix(train_x2), y = train_y, alpha = 0, lambda = penalty_ridge,standardize = FALSE)
glm_lasso_mod <- glmnet(x = as.matrix(train_x2), y = train_y, alpha = 1, lambda = penalty_lasso,standardize = FALSE)
glm_net_mod <- glmnet(x = as.matrix(train_x2), y = train_y, alpha = 0.1, lambda = penalty_net,standardize = FALSE)

```

## Evaluating the Regularized Regression Models' Performance 
The performance accuracy of the models was evaluated by comparing the predictions of each model with the actual sale prices in the training data. The graphs below show the predicted vs true Sale Price, and the score is the Root-Mean-Square Error(RMSE), which is the same metric Kaggle uses in its prediction evaluation for prediction submissions. The smaller the RMSE value the better, and the closer the dots are to the red dashed line, the closer the predicted values are to the actual values (this is based the on training data-set).

```{r, reg_predict_train, echo=FALSE}
## Predict on training set to evaluate the model performance 
y_pred_ridge <- as.numeric(predict(glm_ridge_mod, as.matrix(train_x2)))
y_pred_lasso <- as.numeric(predict(glm_lasso_mod, as.matrix(train_x2)))
y_pred_net<- as.numeric(predict(glm_net_mod,as.matrix(train_x2)))
```

```{r, plot_pred_ridge, fig.cap="Ridge Fit" }
plot(y_pred_ridge,train_y)
abline(a=0, b=1, col="red", lwd=3, lty=2)
```

```{r, plot_pred_lasso, fig.cap="Lasso Fit" }

plot(y_pred_lasso,train_y)
abline(a=0, b=1, col="red", lwd=3, lty=2)

```

```{r plot_pred_net, fig.cap="Elastic Net Fit"}
plot(y_pred_net,train_y)
abline(a=0, b=1, col="red", lwd=3, lty=2)

```

The three regression models Lasso, Ridge, and Elastic Net performed similarly during training, with RMSE of about 0.11. The choice for best model is the Elastic Net model, because it includes fewer features, so it is a simpler model. 

```{r, echo=FALSE}

#Ridge
ridge_SSE <- sum((y_pred_ridge - train_y)^2)
ridge_SST <- sum((train_y - mean(y_pred_ridge))^2)
ridge_R_squared <- 1 - ridge_SSE / ridge_SST
ridge_corr<-cor(y_pred_ridge, train_y)
ridge_RMSE<-rmse(train_y, y_pred_ridge)

#Lasso
lasso_SSE <- sum((y_pred_lasso - train_y)^2)
lasso_SST <- sum((train_y - mean(y_pred_lasso))^2)
lasso_R_squared <- 1 - lasso_SSE / lasso_SST
lasso_corr<-cor(y_pred_lasso, train_y)
lasso_RMSE<-rmse(train_y,y_pred_lasso)

# Elastic Net

net_SSE <- sum((y_pred_net - train_y)^2)
net_SST <- sum((train_y - mean(y_pred_net))^2)
net_R_squared <- 1 - net_SSE / net_SST
net_corr<-cor(y_pred_net, train_y)
net_RMSE<-rmse(train_y,y_pred_net)

#Summary Table
regression_models<- c("Ridge", "Lasso", "Elastic Net")
tunning_parameters<-c(penalty_ridge, penalty_lasso, penalty_net)
nfeatures<-c(rfeatures, lfeatures,efeatures )
R_Squared<-c(ridge_R_squared, lasso_R_squared, net_R_squared)
Cor<- c(ridge_corr, lasso_corr, net_corr)
RMSE<-c(ridge_RMSE, lasso_RMSE, net_RMSE)

kable(data.frame(Models=regression_models, Parameters= tunning_parameters, Features=nfeatures, Rsquared=R_Squared, Corr=Cor,  RMSE = RMSE))

```

Among the top features selected by the Elastic Net model, are several of the features created during the feature engineering phase. For example, the Total Area,  Neighborhodd Category, Overall Score, Total Baths, and Age Category are in the top most important features, and all of these were the result of feature engineering. 

```{r, net_features, echo=FALSE, fig.cap="Elastic Net Features"}
vip(glm_net_mod, num_features = 30, geom = "point",aesthetics = list(color = "Blue"))+ theme_light()+ggtitle("Top 30 Features Ordered by Importance Elastic Net Model")
```

```{r}
lassoVarImp <- varImp(glm_lasso_mod,scale=F, lambda = penalty_lasso)
lassoImportance <- lassoVarImp$importance
varsSelected <- length(which(lassoImportance$Overall!=0))
varsNotSelected <- length(which(lassoImportance$Overall==0))
```

```{r}

vip(glm_ridge_mod, num_features = 20, geom = "point", aesthetics = list(color = "Blue"))+ theme_light()+ggtitle("Top 20 Features Ordered by Importance Ridge Model")

vip(glm_lasso_mod, num_features = 20, geom = "point",aesthetics = list(color = "Blue"))+ theme_light()+ggtitle("Top 20 Features Ordered by Importance Lasso Model")

vip(glm_net_mod, num_features = 20, geom = "point",aesthetics = list(color = "Blue"))+ theme_light()+ggtitle("Top 20 Features Ordered by Importance Elastic Net Model")
```

## Training XGBoost Model with Feature Selection
Next we will implement an advanced machine learning model with a gradient boosted tree model using the xgboost package [@xgboost]. First, we will do some feature selection using the Boruta algorithm from the Boruta package [@Boruta]. 

The Boruta algorithm is a wrapper built around the random forest classification algorithm. It selects the important features  with respect to the outcome variable.

**Feature Selection** 

Boruta selected 65 attributes confirmed important, and 88 attributes confirmed unimportant. After the boruta feature selection, 88 features were removed, living the data-set with 75 columns, the selected features are listed in the next page. 

```{r, FeatureSelctionBoruta, message=FALSE}
# The Boruta algorithm for feature selection uses Random Forest
set.seed(123)        # to get same results
train_boruta <- Boruta(SalePrice~., data=train_1, doTrace=2, maxRuns=125)
# printing the results 
#print(train_boruta)

```
**Selected Features:**

```{r ApplyFeatureSelction,echo=FALSE}
# Getting selected feautres and its stats.
trainb_fix <- TentativeRoughFix(train_boruta)
trainb_selfeat <- getSelectedAttributes(trainb_fix, withTentative = F)
trainb_selfeat_stats <- attStats(trainb_fix)
trainb_selfeat
borutafeatures<-length(trainb_selfeat)
```

This plot shows the selected features in green, and the rejected features in red. The yellow, represent tentative attributes. 

```{r BorutaSelection,echo=FALSE}

#Plot results of Boruta variable selection
plot(train_boruta , las=2, cex.axis=0.6, xlab='', main="Feature Selection")
```

### Training the XGboost Model 

The XGBoost model has several sets of parameters that can be customized: we focused on only two: general parameters, which relate to which booster is used, in this case we selected a 'gbtree'; and the second parameters we tuned were the booster parameters. For the booster parameter, we used 5-fold cross validation to determine the optimal number of rounds. 
For the rest of the booster parameters, the best way to find the optimal parameters is to set a search grid, however this was taking too long (more than an hour to run). So, instead of using this approach I started with the default parameters, and manually changed the min_child_weight(default=1), and  eta(default = 0.3), about three times, until I got a better RMSE on the training data. This is not the ideal way to do it, but I am  not sure my computer was ever going to get through it. 

The final parameters chosen, which resulted in training RMSE:0.125043, were:

\begin{itemize}
  \item booster $=$ gbtree
  \item eta $=$ 0$.$05, default $=$ 0.3, controls the learning rate
  \item max depth $=$ 6, default $=$ 6, tree depth
  \item min child weight $=$ 4, default $=$ 1, minimum number of observations required in each terminal node
  \item subsample $=$ 1
  \item colsample bytree $=$1, percent of columns to sample from for each tree
  \item nrounds $=$ 243
\end{itemize}


```{r, XGB with FeatureSelction,echo=FALSE, message=FALSE }
train_x_fs<-train_1[,trainb_selfeat] #1,460 rows, 75 columns
train_y_fs<-train_1$SalePrice
test_1_fs<-test_1[,trainb_selfeat] #  1,459  75 columns

# Put testing & training data into two seperates Dmatrixs objects
label_train_fs <- train_y_fs

dtrain_fs <- xgb.DMatrix(data = as.matrix(train_x_fs), label= label_train_fs )
dtest_fs <- xgb.DMatrix(data = as.matrix(test_1_fs))

#CV control
my_control_fs <-trainControl(method="cv", number=5)
# xgb grid was taking too long
xgb_grid_sf = expand.grid(
nrounds = 1000,
eta = c(0.1, 0.05, 0.01),
max_depth = c(2, 3, 4, 5, 6),
gamma = 0,
colsample_bytree=1,
min_child_weight=c(1, 2, 3, 4 ,5),
subsample=1
)
```

#### Tuning Paraments
```{r XGB Parameter Tuning, echo=FALSE, message=FALSE }
#Parameter Tuning
default_param_sf<-list(
        booster = "gbtree",
        eta=0.05, #default = 0.3
        gamma=0,
        max_depth=6, #default=6
        min_child_weight=4, #default=1
        subsample=1,
        colsample_bytree=1
)

#Cross validation to determine the best number of rounds 
set.seed(123)

xgbcv_fs <- xgb.cv( params = default_param_sf, data =dtrain_fs, nrounds = 500, nfold = 5, showsd = T, stratified = T, print_every_n_fs = 40, early_stopping_rounds = 10, maximize = F)
xgbcv_fs$best_iteration
cat("The XGB model includes",  borutafeatures,  "features, and  the optimal number of rounds:",  xgbcv_fs$best_iteration, "was selected based on  RMSE of 0.121172")

```


```{r train the model , echo=FALSE, message=FALSE }

xgb_mod_fs <- xgb.train(data = dtrain_fs, params=default_param_sf, nrounds = xgbcv_fs$best_iteration)
xgb_mod_fs 

```

## Evaluating XGBoost Model's Performance

The XGB model performs much better (with R-squared=0.9831584) than the best performing regularized regression model: Lasso (R-squared=0.9226179). So, 98% of the variation in the in SalePrice is explained by the XGB model vs only 92% by the Lasso model (similarly with the Ridge and Elastic Net models).

```{r, XGBSave,echo=FALSE}

#Calculate RMSE on training data
XGBpred_train <- predict(xgb_mod_fs, dtrain_fs)

XGB_SSE <- sum((XGBpred_train - train_y_fs)^2)
XGB_SST <- sum((train_y_fs - mean(XGBpred_train))^2)
XGB_R_squared <- 1 - XGB_SSE / XGB_SST
XGB_corr<-cor(XGBpred_train, train_y_fs)
XGB_RMSE<-rmse(train_y_fs,XGBpred_train)


regression_models<- c("Ridge", "Lasso", "Elastic Net", "XGBoost")
tunning_parameters<-c(penalty_ridge, penalty_lasso, penalty_net, " ")
nfeatures<-c(rfeatures, lfeatures, efeatures, borutafeatures)
R_Squared<-c(ridge_R_squared, lasso_R_squared, net_R_squared, XGB_R_squared)
Cor<- c(ridge_corr, lasso_corr, net_corr, XGB_corr)
RMSE<-c(ridge_RMSE, lasso_RMSE, net_RMSE, XGB_RMSE)
kable(data.frame(Models=regression_models, Parameters= tunning_parameters, Features=nfeatures,Corr=Cor,  Rsquared=R_Squared, RMSE = RMSE))


```
 
# Prediction

## Prediction with Regularized Regression Models

```{r reg_predict}
pred_ridge_test <-  as.double(predict(glm_ridge_mod, as.matrix(test_x2)))
pred_lasso_test <-  as.double(predict(glm_lasso_mod, as.matrix(test_x2)))
pred_net_test<-  as.double(predict(glm_net_mod,as.matrix(test_x2)))
pred_ridge_test_final<- exp(pred_ridge_test)
pred_lasso_test_final<- exp(pred_lasso_test)
pred_net_test_final<- exp(pred_net_test)
head(pred_net_test_final)
```

## Prediction with XGboost Model 
```{r xgb_predict,}
#Prediction
XGBpred_fs <- predict(xgb_mod_fs, dtest_fs)
predictions_XGB_fs <- exp(XGBpred_fs) #reverse the log to the real values
head(predictions_XGB_fs)
```

**Store the item IDs and class labels in a csv file**

```{r savepreds,}

Ridge_model_final_data2<- data.frame("ID"=test_final$Id,"SalePrice"=pred_ridge_test_final)
write.csv(Ridge_model_final_data2, "Ridge_ModelFinal2", row.names=FALSE)

Lasso_model_final_data2<- data.frame("ID"=test_final$Id,"SalePrice"=pred_lasso_test_final)
write.csv(Lasso_model_final_data2, "Lasso_ModelFinal2", row.names=FALSE)

ElasticNet_model_final_data2<- data.frame("ID"=test_final$Id,"SalePrice"=pred_net_test_final)
write.csv(ElasticNet_model_final_data2, "ElasticNet_ModelFinal2", row.names=FALSE)

xgbmodel_pred_d2_fs<- data.frame("ID"=test_ID,"SalePrice"=predictions_XGB_fs)
write.csv(xgbmodel_pred_d2_fs, "xgbModel_fs_outl", row.names=FALSE) 

```

# Validation

### Kaggle Submission
The table below shows how the models performed in the final prediction. The best performing model in the Kaggle prediction is the XGboost model, after feature selection was performed. The regularization models performed slightly better on training than on Kaggle, with RMSE or around .12 on training and .13 on Kaggle final prediction. This indicates that the models are still slightly over-fitted. The XGB model, which is the best model based on the prediction RMSE performed worse on the Kaggle prediction with RMSE of .12758 than it did on training. This model is probably over-fitted, and fine tuning the  model's parameters with an expanded grid search to find the optimal tuning parameters would probably result in a better model.  

Below are the final models submitted to Kaggle.com, and the corresponding Kaggle score. 


**Final Summary Table**

```{r, echo=FALSE}

models<- c("Ridge", "Lasso", "Elastic Net", "XGBoost")
tunning_parameters<-c(penalty_ridge, penalty_lasso, penalty_net, " ")
nfeatures<-c(rfeatures, lfeatures, efeatures, borutafeatures)
R_Squared<-c(ridge_R_squared, lasso_R_squared, net_R_squared, XGB_R_squared)
Cor<- c(ridge_corr, lasso_corr, net_corr, XGB_corr)
RMSE<-c(ridge_RMSE, lasso_RMSE, net_RMSE, XGB_RMSE)
KaggleRMSE<-c(0.13163, 0.12998, 0.13061, 0.12758 )
kable(data.frame(Models=models, Features=nfeatures,Corr=Cor,  Rsquared=R_Squared, TrainingRMSE = RMSE, KaggleRMSE=KaggleRMSE))

```

# Conclusion
## Answer to Research Questions

1. Which are the most relevant features that impact sales price?

Based on the best performing model, the following features are the top 20 predictors of SalesPrice for the given data. 


```{r}
feature_importance_fs <- xgb.importance (feature_names = colnames(dtrain_fs),model = xgb_mod_fs)
#xgb.plot.importance(importance_matrix = feature_importance_fs[1:20])
xgb.ggplot.importance(importance_matrix = feature_importance_fs[1:20], rel_to_first = TRUE)+ggtitle("XGB Top 20 Features")

```
^[The plot was produced with the package 'vip' [@vip].]

The most important features for the XGB model were Total Area, Overall Quality,  Neighborhood Category, Total Baths and Overall Score. Also, Fireplace Quality, Lot Area, Exterior Quality, GarageCars, Garage Finish, Garage Grade and Central Air had significant impact on sales price. 

So, what does this mean?

For example, Central Air seems to be an important predictor of price, so if you are a buyer you could consider getting a house without central and installing it yourself after the purchase. On the other hand if you were a seller do the opposite to fetch a better sale price. The basement finish quality seems to be important, so you could consider saving by getting an unfinished basement if you were a buyer, and finish the basement if you were a seller. Also the fire place quality had significant impact on price, so as a buyer maybe consider a fireplace that needs some work, while as a seller fix up the fire place before selling, and the same goes for the quality of the garage. 

2. Which pre-processing techniques improve the predictive value of the data?
Data Cleaning, data engineering, near zero variance filtering, normalizing the response, normalizing and standardizing the numerical features were the pre-processing techniques used. 

The feature engineering definitely paid off since a few of our transformed features are right up in the top 20 on level of importance at predicting the sales price of homes. Combining the total square footage was successful as was combining the total numbers of baths, total basement square-footage, porch square-footage were also successful (in the top 20).  Bining the neighborhoods definitely had an impact since neighborhood category is in the top three predictors for the models. The age category also is in the top 20. 
By performing feature engineering, feature selection, regularization, and transforming the target variable,  we can reduce loss and greatly improve the performance.

Also, the model seems to perform better without removing the 4 outliers in the training set. With outliers removed the best score in Kaggle RMSE is 0.12758, while without removing the outliers the best Kaggle RMSE is 0.12592. These values are for the XGB model, but the results are similar for all the other models attempted here. 

3. Which model performs best at predicting price?

Overall the XGB model performed better both in training and in the final Kaggle prediction score.

4. Which approach in dealing with multicollinearity yields a better prediction?
The XGBoost model with feature selection performed better at predicting price than the regularized regression models. 

Finally, in the quest for a better score in this addictive Kaggle competition, I tried something I had read other competitors were doing to improve their scores. They say, that if you average the top performing model's predictions, you can get a better prediction score on Kaggle.com's competitions. I tried it, by averaging the predictions from the XGB, Elastic Net, Lasso and GBM models, and it worked.  This trick got me into the top 19%, with an RMSE score of 0.12595. 

```{r}
stacking <- data.frame("ID"=test_ID,"SalePrice"=(predictions_XGB_fs+pred_net_test_final+pred_lasso_test_final)/3)
head(stacking)
write.csv(stacking , "stacked_fs_outl", row.names=FALSE) #KAGGLE RMSE: 0.12592

```


## Future Work

Two areas that could be improved is the outlier analyis in the dataset and the search grid for the XGB model. 

# Bibliography
