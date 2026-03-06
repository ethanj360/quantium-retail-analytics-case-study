---
title: "Task 1"
output: pdf_document
---
```{r}
#Start Section/Chunk
#installing packages
#install.packages(c("data.table", "readr", "ggplot2", "stringr"), dependencies = TRUE)

#loading packages
library(data.table)
library(readr)
library(ggplot2)
library(stringr)

# getwd checks folder name, list.files checks files in folder and their names
getwd()
list.files()

# load data. create name/variable to store transaction and customer data set after loading it
transactionData <- fread("QVI_transaction_data.csv")
customerData <- fread("QVI_purchase_behaviour.csv")

# confirm the datasets are loaded. shows dimensions IE number of columns and rows
dim(transactionData)
dim(customerData)
```


```{r}
#Data Cleaning Chunk/ Section
# quick structure checks. checking column names + types
### commented out after checking 


names(transactionData)
# str(transactionData)

names(customerData)
# str(customerData)
#result: because date is INT, need to change to DATE

# stats for quantities of chips purchased IE min, max, median, 1st quartile (25th percentile) and 3rd quartile (75th percentile) to help spot outliers
#PROD_QTY shows how many chip packages bought in transaction
summary(transactionData$PROD_QTY)

# stats for sales of chips IE min, max, median, 1st quartile (25th percentile) and 3rd quartile (75th percentile) to help spot outliers
#TOT_SALES shows dollar amount for that line item
summary(transactionData$TOT_SALES)

# counting missing values (NA counts by column)
colSums(is.na(transactionData))
colSums(is.na(customerData))
#result: all 0s, so it is good



#FIX DATES
# change Date from INT into DATE
#1899-12-30 is origin date used by Excel-style date numbers
transactionData$DATE <- as.Date(transactionData$DATE, origin = "1899-12-30")

# confirm conversion worked
summary(transactionData$DATE)




#CHECK TO MAKE SURE ALL PRODUCTS ARE CHIPS
#show 20 product names
head(transactionData$PROD_NAME, 20)
#Result: found product named "Old El Paso Salsa", meaning i have to find and remove any products that aren't chips

# make product names lowercase (easier to search)
prod_lower <- tolower(transactionData$PROD_NAME)

# define non-chip keywords
non_chip_terms <- c("salsa", "dip")

# mark rows that contain any of these words
transactionData$NON_CHIP <- grepl(paste(non_chip_terms, collapse="|"), prod_lower)

# see how many rows will be removed
sum(transactionData$NON_CHIP)

# show unique product names that are flagged as NON_CHIP (limit 20 so result is not too long)
head(unique(transactionData$PROD_NAME[transactionData$NON_CHIP])[1:20])

prod_lower <- tolower(transactionData$PROD_NAME)

# names that contain "dip" (changed to only 10 to reduce pdf size)
head(unique(transactionData$PROD_NAME[grepl("dip", prod_lower)])[1:10])

# names that contain "salsa" but NOT "dip" (possible salsa-flavored chips)(changed to only 10 to reduce pdf size)
head(unique(transactionData$PROD_NAME[grepl("salsa", prod_lower) & !grepl("dip", prod_lower)])[1:10])


#remove NA from results
prod_lower <- tolower(transactionData$PROD_NAME)

salsa_not_dip_names <- unique(transactionData$PROD_NAME[
  !is.na(prod_lower) &
  grepl("salsa", prod_lower) &
  !grepl("dip", prod_lower)
])

salsa_not_dip_names
#result: these are mostly salsa chips, not dip. need to remove only the oners that are dip 


# lowercase version for matching
prod_lower <- tolower(transactionData$PROD_NAME)

#flag anything with the word "dip" or "dips" in the product name
#The pattern \\bdips?\\b means match the word dip or dips:
#dip is the word
#s? means “the s is optional” (so dip or dips)
#\\b means “word boundary” so it matches the whole word, not part of another word

is_dip_word <- !is.na(prod_lower) & grepl("\\bdips?\\b", prod_lower)

#flag the Woolworths salsa items that are likely dip
is_woolworths_salsa <- !is.na(prod_lower) &
  grepl("woolworths", prod_lower) &
  grepl("salsa", prod_lower)

# combine both rules
transactionData$NON_CHIP <- is_dip_word | is_woolworths_salsa

# review what will be removed (first 50 unique names) (changed to 10 later on to reduce PDF size)
head(unique(transactionData$PROD_NAME[transactionData$NON_CHIP])[1:10])

# remove those rows
transactionData <- transactionData[transactionData$NON_CHIP == FALSE, ]

# clean up helper column
transactionData$NON_CHIP <- NULL

```


```{r, message=FALSE, warning=FALSE}
#checking after cleaning
dim(transactionData)
summary(transactionData$PROD_QTY)
summary(transactionData$TOT_SALES)


#remove outlier customer quantity as is says someone purchased 200 chip packages
outlier_cards <- unique(transactionData$LYLTY_CARD_NBR[transactionData$PROD_QTY == 200])
outlier_cards

transactionData <- transactionData[!transactionData$LYLTY_CARD_NBR %in% outlier_cards, ]


#check to see if max quantity is normal
max(transactionData$PROD_QTY)

```




```{r, results='hide'}
#CHECKING DATES
# Count number of transaction lines per day
transactions_by_day <- as.data.table(transactionData)[, .N, by = DATE][order(DATE)]

# How many unique dates do we have?
nrow(transactions_by_day)

# Create a complete date sequence based on min/max in your data
all_dates <- data.table(DATE = seq(min(transactionData$DATE), max(transactionData$DATE), by = "day"))

# Join to find missing days (days with zero transactions)
transactions_by_day_full <- merge(all_dates, transactions_by_day, by = "DATE", all.x = TRUE)
transactions_by_day_full[is.na(N), N := 0]

# List missing/zero-transaction days (if any)
transactions_by_day_full[N == 0]

#result shows that Christmas 2018 is missing in this data set. this is nice to note on the report


#plot for transactions over time 
ggplot(transactions_by_day_full, aes(x = DATE, y = N)) +
  geom_line() +
  labs(title = "Transaction Lines Over Time", x = "Date", y = "Transactions") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```





```{r}
#Create Pack_Size + Brand
#Create pack size (grams) from product name
transactionData$PACK_SIZE <- readr::parse_number(transactionData$PROD_NAME)

# create brand as first word of product name
transactionData$BRAND <- stringr::word(transactionData$PROD_NAME, 1)

#Red Rock Deli shows as RRD at times, but also shows as "RED" because taking first word of product name. change so that shows as 1 brand instead of RED and RRD as seperate brands
transactionData$BRAND[transactionData$BRAND == "RED"] <- "RRD"

# check a few rows
head(transactionData[, c("PROD_NAME", "PACK_SIZE", "BRAND")], 5)

# top brands
head(sort(table(transactionData$BRAND), decreasing = TRUE), 15)
```

```{r}
# Merge customer segments onto each transaction (by loyalty card number)
# For every transaction, look up the customer’s segment info using the loyalty card number, and attach it to the transaction
data <- merge(transactionData, customerData, by = "LYLTY_CARD_NBR", all.x = TRUE)

# check that merge worked: should be same row count as transactionData
dim(transactionData)
dim(data)

# check for missing segment info after merge (should be 0,0)
colSums(is.na(data[, c("LIFESTAGE", "PREMIUM_CUSTOMER")]))



# Time to do analysis
# Total sales by segment
sales_by_seg <- aggregate(TOT_SALES ~ LIFESTAGE + PREMIUM_CUSTOMER, data = data, sum)

# Number of customers by segment
cust_by_seg <- aggregate(LYLTY_CARD_NBR ~ LIFESTAGE + PREMIUM_CUSTOMER, data = data,
                         function(x) length(unique(x)))
names(cust_by_seg)[3] <- "N_CUSTOMERS"

# Units per customer by segment
units_by_seg <- aggregate(PROD_QTY ~ LIFESTAGE + PREMIUM_CUSTOMER, data = data, sum)
units_by_seg <- merge(units_by_seg, cust_by_seg, by = c("LIFESTAGE", "PREMIUM_CUSTOMER"))
units_by_seg$UNITS_PER_CUST <- units_by_seg$PROD_QTY / units_by_seg$N_CUSTOMERS

# Avg price per unit by segment
price_by_seg <- aggregate(cbind(TOT_SALES, PROD_QTY) ~ LIFESTAGE + PREMIUM_CUSTOMER,
                          data = data, sum)
price_by_seg$AVG_PRICE_PER_UNIT <- price_by_seg$TOT_SALES / price_by_seg$PROD_QTY

# show top segments by total sales
sales_by_seg_sorted <- sales_by_seg[order(-sales_by_seg$TOT_SALES), ]
head(sales_by_seg_sorted, 10)

```

```{r}
# plots
#total sales
ggplot(sales_by_seg, aes(x = LIFESTAGE, y = TOT_SALES, fill = PREMIUM_CUSTOMER)) +
  geom_col(position = "dodge") +
  labs(title = "Total Chip Sales by Customer Segment", x = "Lifestage", y = "Total Sales") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))



#number of customers
ggplot(cust_by_seg, aes(x = LIFESTAGE, y = N_CUSTOMERS, fill = PREMIUM_CUSTOMER)) +
  geom_col(position = "dodge") +
  labs(title = "Number of Customers by Segment", x = "Lifestage", y = "Customers") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))



#units per customer
ggplot(units_by_seg, aes(x = LIFESTAGE, y = UNITS_PER_CUST, fill = PREMIUM_CUSTOMER)) +
  geom_col(position = "dodge") +
  labs(title = "Average Units per Customer by Segment", x = "Lifestage", y = "Units per Customer") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))



#average price per unit
ggplot(price_by_seg, aes(x = LIFESTAGE, y = AVG_PRICE_PER_UNIT, fill = PREMIUM_CUSTOMER)) +
  geom_col(position = "dodge") +
  labs(title = "Average Price per Unit by Segment", x = "Lifestage", y = "Avg Price per Unit") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))


```


```{r}

# Top 5 segments by TOTAL SALES
head(sales_by_seg_sorted, 5)

# Top 5 segments by NUMBER OF CUSTOMERS
cust_by_seg_sorted <- cust_by_seg[order(-cust_by_seg$N_CUSTOMERS), ]
head(cust_by_seg_sorted, 5)

# Top 5 segments by UNITS PER CUSTOMER
units_by_seg_sorted <- units_by_seg[order(-units_by_seg$UNITS_PER_CUST), ]
head(units_by_seg_sorted, 5)

# Top 5 segments by AVG PRICE PER UNIT
price_by_seg_sorted <- price_by_seg[order(-price_by_seg$AVG_PRICE_PER_UNIT), ]
head(price_by_seg_sorted, 5)

```

## Key insights
- Sales are highest for Older Families (Budget), Young Singles/Couples (Mainstream), and Retirees (Mainstream).
- Young Singles/Couples (Mainstream) and Retirees (Mainstream) have the most customers, which helps explain their high sales.
- Family segments (Older Families, Young Families) buy the most units per customer, so their sales are more volume-driven.
- Mainstream singles/couples pay a higher average price per unit, suggesting a willingness to pay for preferred brands or flavors.

## Recommendation to Julia
- Focus growth efforts on Older Families (Budget) with volume offers (multi-buy, larger packs) since they purchase more units per customer.
- For Young Singles/Couples (Mainstream), prioritize popular brands and flavor variety and run occasion based promotions (weekends, gatherings) since this segment has high customer count and higher price per unit.
- Maintain core range availability for Retirees (Mainstream) to protect a large customer base and steady sales.

