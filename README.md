# Price-optimzation

include = F
# install.packages("readxl")
# install.packages("forecast")
# install.packages("tseries")
# install.packages("Metrics")
# install.packages("vars")
# install.packages("tsDyn")
# install.packages("DAAG")
# install.packages("lubridate")
# install.packages("caret")
# install.packages("randomForest")
# install.packages("xlsx")
suppressWarnings(library("readxl",verbose =F))
suppressWarnings(library("forecast",verbose =F))
suppressWarnings(library("tseries",verbose =F))
suppressWarnings(library("Metrics",verbose =F))
suppressWarnings(library("vars",verbose =F))
suppressWarnings(library("lubridate",verbose =F))
suppressWarnings(library("caret",verbose =F))
suppressWarnings(library("tsDyn",verbose =F))
#library("DAAG")
suppressWarnings(library("randomForest",verbose =F))
suppressWarnings(library("xlsx",verbose =F))
data <- read_xlsx("Test.xlsx")
data$DayOfWeekNumber <- as.factor(data$DayOfWeekNumber)

# Data preprocessing

# tranforming data and converting to time series object to apply time series model
combine<- log(data[,5:8])
comb_ts <- ts(combine, start = c(2016,11), frequency = 7)
comb_df <- data.frame(comb_ts)

# plotting time series graphs
plot.ts(comb_df$WtAvgEffPrice)
plot.ts(comb_df$Quantity.Sold)

# checking if data is stationary for time series
adf.test(comb_df$WtAvgEffPrice)
adf.test(comb_df$Quantity.Sold)

# plotting acf and pacf plot to check autocorrelation 
acf(comb_df$WtAvgEffPrice)
pacf(comb_df$WtAvgEffPrice)
acf(comb_df$Quantity.Sold)
pacf(comb_df$Quantity.Sold)
# It is clearly seen from plots that there is high correlation on lag 7, 14, 21 so on hence 
# this data has weekly seasonality which will be considered while building time series model

# Partitioning data into train and test
train_comb <- comb_df[1:330,]
test_comb <- comb_df[331:367,]
train_data <- data[1:330,]
test_data <- data[331:367,]

# VAR model to forecast test data
var1 <- VAR(train_comb,p = 2,type = "both", lag.max = 7, season = 7)

#summary(var1)
pred <- predict(var1, n.ahead = 37)
pred_df <- as.data.frame(pred$fcst)
pred_actual <- exp(pred_df[,c(1,5,9,13)])

# Checking accuracy of VAR model on test data
forecast::accuracy(pred_actual$Quantity.Sold.fcst, test_data$`Quantity Sold`)
forecast::accuracy(pred_actual$EffectivePriceRevenue.fcst, test_data$EffectivePriceRevenue)
forecast::accuracy(pred_actual$DailyShoeRental.fcst, test_data$DailyShoeRental)
forecast::accuracy(pred_actual$WtAvgEffPrice.fcst, test_data$WtAvgEffPrice)

# VECM model to forecast test data
VECmodel <-  VECM(train_comb,lag = 7, r = 3,include = "const",estim="ML")
#summary(VECmodel)
pred <- predict(VECmodel, n.ahead = 37)
pred_df <- as.data.frame(pred)
pred_actual <- exp(pred_df[,c(1:4)])
forecast::accuracy(pred_actual$Quantity.Sold, test_data$`Quantity Sold`)
forecast::accuracy(pred_actual$EffectivePriceRevenue, test_data$EffectivePriceRevenue)
forecast::accuracy(pred_actual$DailyShoeRental, test_data$DailyShoeRental)
forecast::accuracy(pred_actual$WtAvgEffPrice, test_data$WtAvgEffPrice)
# From both model results it is seen that accuarcy of VAR model is better. I have also tried other 
# time series models like ARIMA, ETS and Holtwinters (code not included for those models). But VAR  
# model provided better results among all. Hence I have selected VAR model to forecast data for next 
# 365 days. 

# 1. Forecasting data for next 365 days using VAR model
combine_df <- ts(comb_ts, start = c(2016,11), frequency = 7)
var2 <- VAR(combine_df,type = "const", lag.max = 7, season = 7)
#summary(var2)
prediction <- predict(var2, n.ahead = 365)
prediction_df <- as.data.frame(prediction$fcst)
predicted_value <- exp(prediction_df[,c(1,5,9,13)])
predicted_value$Quantity.Sold.fcst <- round(predicted_value$Quantity.Sold.fcst)
predicted_value$DailyShoeRental.fcst <- round(predicted_value$DailyShoeRental.fcst)
#range(data$EffectivePriceRevenue)
dates <- seq(as.Date("2017-11-05"), as.Date("2018-11-04"), by = "day")
date_df <- data.frame(dates)
date_df$dayofweeknumber <- wday(date_df$dates)
forecasting_VAR <- cbind(date_df,predicted_value)
colnames(forecasting_VAR)[3:6] <- c("Quantity Sold", "Net Revenue", "Net Rate", "Daily Shoe Rental")
head(forecasting_VAR, 5)

# 2. Optimizing net rate an net revenue using linear algorithm
l1 <- lm(forecasting_VAR$`Net Rate` ~ forecasting_VAR$`Quantity Sold`, data = forecasting_VAR)
l1$coefficients
ctrl <- trainControl(method = "cv", number = 10 )
lmcv <- train(`Net Rate` ~ `Quantity Sold`, data = forecasting_VAR, method = 'lm', trControl = ctrl,
              metric= 'Rsquared')
# summary(lmcv)
Optimized_rate <- NULL
Optimized_rate_all <- NULL
Optimized_revenue <- NULL
Optimized_revenue_all <- NULL
for(i in 1:nrow(forecasting_VAR)) {
Optimized_rate <-  l1$coefficients[1] + l1$coefficients[2] * forecasting_VAR$`Quantity Sold`[i]
Optimized_rate_all <- as.data.frame(rbind(Optimized_rate_all, Optimized_rate))
colnames(Optimized_rate_all)[1] <- c("rates")
Optimized_revenue <- Optimized_rate_all$rates[i] * forecasting_VAR$`Quantity Sold`[i]
Optimized_revenue_all <- as.data.frame(rbind(Optimized_revenue_all, Optimized_revenue))
}
Optimization <- cbind(Optimized_rate_all$rates, Optimized_revenue_all)
rmse(Optimization$`Optimized_rate_all$rates`, forecasting_VAR$`Net Rate`)

# Both normal lm and cv methods are giving same results. hence I have calculated net rate using 
# intercept of quantity sold and constant of equation. I have also calculated revenue through net rate 
# and quantity sold


# Exporting forecasted and optimized values to excel
# write.xlsx(forecasting_VAR, "Forecasted_values.xlsx")
# write.xlsx(Optimization, "Optimized_values.xlsx")

