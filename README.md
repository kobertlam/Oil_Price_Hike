# Energy ETF RYE Forecast

## Overview 

### Background

Brent crude spot price jumped to $133 a barrel on March 8, 2022 and it represents the highest oil prices since 2008. Oil companies supply billions of barrels of petroleum products daily to power transportation and industry. The fluctuation of the crude oil price has direct impact on the production cost of the carbon-based fuels and products, which in turn impacting the profitability and the stock prices of the oil companies. In this project, we would like bring insights to fund managers and investors who are interested in investing the oil-related sector. We will study if there is any relationship between the crude oil price and the Energy ETF (RYE).

### Purpose of this Project

This is the final project for Data Analytics Bootcamp for our group (Group 4). We have integrated the data analystic skills and tools we have learnt from the bootcamp into this final project. We aim to, based on the historical prices of an Exchanged Traded Fund (ETF) in the Energy sector, forecast the future prices of the ETF prices (Invesco S&P 500® Equal Weight Energy ETF). 

### Questions to be Answered

1. Are there seasonal trends and patterns on the Energy ETF? Can we forecast the future ETF prices solely based on the historical ETF prices (time series)?
2. Is there any relationship between the crude oil prices and the ETF prices? Can we forecast the future ETF price based on both the historical ETF prices and the historical crude oil prices?

### Benefits

The value investment theories indicate that the market/nominal stock value of any company tends to approach the real value. If a stock is under-valued for now, the stock price tends to increase. If a stock is over-valued for now, the stock price is likely to decrease over time. 

If we are able to determine the relationship between the stock prices of the oil companies and the crude oil prices, investors or fund managers will be able to make better decisions when analyzing the stock market. 

### Team Collaboration - Communication Protocols

* We created a private group chat in `Slack` as the primary communication channel within the team.
* We also used Zoom meeting for group collaboration.

### Presentation

We prepared a [Google Slide](https://docs.google.com/presentation/d/1M9gE1Wv08GLSOgKtwCtLHypvGdmYd1BzGTxuklBDWRo/edit?usp=sharing) for presentation purposes.

Details on our presentation of the project can also be found in the [presentation](https://github.com/kobertlam/Oil_Price_and_Stock_Price_Analysis/tree/presentation) branch.

## Database

### Source of data

1. Dataset for [Brent Spot Price of Crude Oil](https://www.eia.gov/dnav/pet/hist_xls/RBRTEd.xls) (Brent Spot Price, dollars per Barrel) from U.S. Energy Information Administration
2. ETF price from [`yfinance`](https://pypi.org/project/yfinance/) Yahoo! Finance's API for ticker `RYE` - Invesco S&P 500 Equal Weight Energy ETF
3. ETF portfolio holdings from [Invesco](https://www.invesco.com/us/financial-products/etfs/holdings?audienceType=Investor&ticker=RYE) for this ETF

### Database application

We've decided to use PostgreSQL as our database, as it is easy and efficient for us to connect with our [Jupyter Notebook](../database/postgresql_connection.ipynb). 

Below shows the Entity Relationship Diagram of our database. 

![](../database/Resources/ERD.png)

### Connect with Jupyter Notebook

Below shows the code to set up and connect to the database. 

```
from sqlalchemy import create_engine
from config import db_password

db_string = f'postgresql://postgres:{db_password}@127.0.0.1:5432/energy_etf_rye_forecast'
engine = create_engine(db_string)
db_connection = engine.connect()
```

Below is an example to export the Pandas DataFrame to our database. 

```
rye.to_sql('rye', engine)
brent.to_sql('brent', engine)
```

Below is an example to import the data from our database into Pandas DataFrame. 

```
query = 'SELECT * FROM rye'
rye = pd.read_sql(query, db_connection, parse_dates=['date'], index_col='date')
print('rye shape:', rye.shape)

query = 'SELECT * FROM brent_spot_price_crude_oil'
brent = pd.read_sql(query, db_connection, parse_dates=['date'], index_col='date')
print('brent shape:', brent.shape)

query = 'SELECT rye.*, brent_spot_price_crude_oil.brent FROM rye JOIN brent_spot_price_crude_oil ON rye.date = brent_spot_price_crude_oil.date'
model_df = pd.read_sql(query, db_connection, parse_dates=['date'], index_col='date')
print('model_df shape:', model_df.shape)
```

## Machine Learning Models

### Data preprocessing

After acquiring our datasets, we have imported them into our Jupyter Notebook, conducted some preprocessing, and completed some preliminary analysis. 

Below shows the shapes of our tables. With 'date' to be the index, we have 'open', 'high', 'low', 'close', 'adj close', 'volume', and 'brent' as columns. We have 3,872 rows for the RYE historical trading data and 8,845 rows for the Brent Spot Price of Crude Oil data. After joining the tables together, we have a total of 3,842 valid rows. The date range of our data is from November 7, 2006 to March 21, 2022. 

```
rye shape: (3872, 6)
brent shape: (8845, 1)
model_df shape: (3842, 7)
```

#### Preliminary analysis

Below shows the historical daily close prices of the Invesco S&P 500 Equal Weight Energy ETF, which is an exchange traded fund with a portfolio of companies in the energy sector. 

![](../machine_learning_model/Resources/rye_daily.png)

The data is also visualized below through a probability distribution. 

![](../machine_learning_model/Resources/rye_probability_distribution.png)

#### Stationarity analysis

Commonly, a given time series consist of three systematic components and one non-systematic component. The three systematic components include Level, Trend, and Seasonality. The one non-systematic component is the noise. 

> - Level: the average value in the series
> - Trend: the increasing or decreasing value in the series
> - Seasonality: the repeating short-term cycle in the series
> - Noise: the random variation in the series

Before further analysis, we have conducted the [Augmented Dickey-Fuller Test](https://en.wikipedia.org/wiki/Augmented_Dickey%E2%80%93Fuller_test) to check if our time series data is stationary or not, as the time series analysis only works with stationary data. 

The ADF test is one of the most popular statistical tests used to determine the presence of a unit root in the time series, which helps identify if a time series is stationary or not. The null and alternate hypotheses are the following. If we fail to reject the null hypothesis, we can say that the time series is non-stationary, which means this time series can be linear or difference stationary. If both the mean and the standard deviation lines are flat, which implies that the mean and the variance are constant, the time series are stationary. 

- Null Hypothesis: the series has a unit root (value of a = 1)
- Alternate Hypothesis: the series has no unit root

Below are the results and the visualization. We can see that the time series is not stationary. The p-value is 0.220827, greater than 0.05, so we are not able to reject the Null Hypothesis. In addition, the Test Statistics are greater than the critical values. From the graph, we are able to see that the mean and the variance are fluctuating as well.

```
Results of dickey fuller test
Test Statistics                  -2.160673
p-value                           0.220827
No. of lags used                  3.000000
Number of observations used    3868.000000
critical value (1%)              -3.432042
critical value (5%)              -2.862288
critical value (10%)             -2.567168
dtype: float64
```

![](../machine_learning_model/Resources/rye_stationarity.png)

#### Separate trend and seasonality

In order to perform a time series analysis, we need to separate trend and seasonality from the time series. The resultant series will become stationary through this process. 

![](../machine_learning_model/Resources/rye_seasonality.png)

The `seasonal_decompose` function has been used to take a log of the time series to reduce the magnitude of the values and reduce the rising trend. Then, we find the rolling average of the time series. A rolling average is calculated by taking input for the past 12 months and by giving a mean consumption value at every point further ahead in the time series. 

![](../machine_learning_model/Resources/rye_masd.png)

### Feature engineering and selection

Forecasting the price of a financial asset is a complex challenge. In general, the price is determined by a variety of variables, economic cycles, unforeseen events, psychological factors, market sentiment, the weather, or even war. All these variables will more or less have an influence on the price of the financial asset. In addition, many of these variables are interdependent, which makes statistical modeling even more complex. 

A univariate forecast model reduces this complexity to a minimum – a single factor and ignores the other dimensions. A multivariate model is a simplification as well, but it can take several factors into account. For example, a multivariate stock market prediction model can consider the relationship between the closing price and the opening price, moving averages, daily highs, the price of other stocks, and so on. Multivariate models are not able to fully cover the complexity of the market. However, they offer a more detailed abstraction of reality than univariate models. Multivariate models thus tend to provide more accurate predictions than univariate models.

![](../machine_learning_model/Resources/uni_multi.png)

For the multivariate model, we have included open, high, low, close prices, and volume of this ETF as our features. 

![](../machine_learning_model/Resources/rye_multi.png)

Because the exchange traded fund of our selection is an ETF from the energy sector, in addtion to the open, high, low, close prices, and volume of this ETF, we have also included the Brent Spot Price of Crude Oil into our neural networks model. 

![](../machine_learning_model/Resources/sequential_daily.png)

After thorough group discussion, we have decided to proceed with one univariate approach and one multivariate approach. The univariate approach will be using the ARIMA model, to predict the future ETF prices solely based on the historical ETF prices. The multivariate approach will be using the Sequential model in neural networks, taking the open, high, low, close prices, volume, as well as the Brent Spot Price of Crude Oil into consideration. Below shows the features that we are going to use in our multivariate model. 

![](../machine_learning_model/Resources/features.png)

### Data split into training and testing sets

For both models, ARIMA and Sequential, we have split the data into 80% training dataset and 20% testing dataset. 

![](../machine_learning_model/Resources/arima_split.png)

### Models of choice

#### ARIMA

The Auto ARIMA `auto_arima` function is used to identify the most optimal parameters for an ARIMA model, and returns a fitted ARIMA model. This function is based on the commonly-used R function, `forecast::auto.arima`. It works by conducting different tests, including Kwiatkowski–Phillips–Schmidt–Shin, Augmented Dickey-Fuller or Phillips–Perron, to determine the order of differencing, d, and then fitting models within ranges of defined start_p, max_p, start_q, max_q ranges. If the seasonal optional is enabled, it can also identify the optimal P and Q hyper-parameters after conducting the Canova-Hansen to determine the optimal order of seasonal differencing, D. 

As can be seen from below results, the best model that we will proceed with is `ARIMA(0,1,0)`. 

```
Performing stepwise search to minimize aic
 ARIMA(0,1,0)(0,0,0)[0] intercept   : AIC=-15264.893, Time=0.12 sec
 ARIMA(1,1,0)(0,0,0)[0] intercept   : AIC=-15265.603, Time=0.10 sec
 ARIMA(0,1,1)(0,0,0)[0] intercept   : AIC=-15265.689, Time=0.24 sec
 ARIMA(0,1,0)(0,0,0)[0]             : AIC=-15266.893, Time=0.06 sec
 ARIMA(1,1,1)(0,0,0)[0] intercept   : AIC=-15218.317, Time=0.37 sec

Best model:  ARIMA(0,1,0)(0,0,0)[0]          
Total fit time: 0.900 seconds
                               SARIMAX Results                                
==============================================================================
Dep. Variable:                      y   No. Observations:                 3094
Model:               SARIMAX(0, 1, 0)   Log Likelihood                7634.446
Date:                Sun, 27 Mar 2022   AIC                         -15266.893
Time:                        23:21:08   BIC                         -15260.856
Sample:                             0   HQIC                        -15264.725
                               - 3094                                         
Covariance Type:                  opg                                         
==============================================================================
                 coef    std err          z      P>|z|      [0.025      0.975]
------------------------------------------------------------------------------
sigma2         0.0004   4.93e-06     85.256      0.000       0.000       0.000
===================================================================================
Ljung-Box (L1) (Q):                   2.71   Jarque-Bera (JB):              7210.79
Prob(Q):                              0.10   Prob(JB):                         0.00
Heteroskedasticity (H):               0.41   Skew:                            -0.57
Prob(H) (two-sided):                  0.00   Kurtosis:                        10.39
===================================================================================

Warnings:
[1] Covariance matrix calculated using the outer product of gradients (complex-step).
```

![](../machine_learning_model/Resources/arima_auto_arima.png)

#### Neural networks - Sequential

Multivariate models are trained on a three-dimensional data structure. The first dimension is the sequences, the second dimension is the time steps (batches), and the third dimension is the features. Below shows the steps that we have used to transform the multivariate data into a shape that our neural networks model can process during the training. When doing a forecast later on, we will use the same structure. 

![](../machine_learning_model/Resources/transforming.png)

```
Train X shape: (3020, 50, 7)
Train Y shape: (3020,)
Test X shape: (767, 50, 7)
Test Y shape: (767,)
```

Below shows the structure of our Sequential model. 

```
Model: "sequential"
_________________________________________________________________
 Layer (type)                Output Shape              Param #   
=================================================================
 lstm (LSTM)                 (None, 50, 350)           501200    
                                                                 
 lstm_1 (LSTM)               (None, 350)               981400    
                                                                 
 dense (Dense)               (None, 5)                 1755      
                                                                 
 dense_1 (Dense)             (None, 1)                 6         
                                                                 
=================================================================
Total params: 1,484,361
Trainable params: 1,484,361
Non-trainable params: 0
_________________________________________________________________
```

Another important step in our multivariate model is to slice the data into multiple input data sequences with associated target values. Two variables, `sequence_length` and `pred_int` will be defined here with the option to modify for different model training. 

We have used the sliding windows algorithm, which moves a window step by step through the time series data, adding a sequence of `sequence_length` number of data points to the train sequence (input data) with each step. Then, the algorithm stores the target value (close) following this train sequence by `pred_int` positions in a separate target value dataset. This process will repeat itself with the window and the target value pushed further until going through the entire time series data. Eventually, the algorithm will create a dataset containing the train sequences (batches) and their corresponding target values. This process will be applied to both training and testing data. 

![](../machine_learning_model/Resources/batch.png)

The model loss drops quickly until stablized at a lower level, impling that the model has improved throughout the training process. 

![](../machine_learning_model/Resources/sequential_model_loss.png)

### Results

#### ARIMA

Below shows the results of the forecast by our ARIMA model on the test dataset based on 95% confidence level. 

![](../machine_learning_model/Resources/arima_forecast.png)

Due to the huge fluctuation of the ETF prices, the time series model ARIMA does not work very well in forecasting the future prices, which is expected, as there are other far more significant variables that have impact on the prices of this ETF. Below shows some commonly used accuracy metrics to evaluate our forecast results. 

```
Mean Squared Error: 0.13534741545347972
Mean Absolute Error: 0.265734282144839
Root Mean Squared Error: 0.36789593019423267
Mean Absolute Percentage Error: 0.07837232492941332
```

#### Neural networks - Sequential

Below shows the predictions vs actuals from the full time period. 

![](../machine_learning_model/Resources/sequential_predictions_full.png)

Below shows the predictions vs actuals from the test time period only. 

![](../machine_learning_model/Resources/sequential_predictions_test.png)

From our latest run, we have got the following excellent results. The MAPE is 2.41% which means the mean of our predictions deviates from the actual values by 2.55%. The MDAPE is 1.81%, lower than the MAPE, which means there are some outliers among the forecast errors. Half of our forecasts deviate by more than 1.81% while the other half by less than 1.81%. 

```
Median Absolute Error (MAE): 0.88
Mean Absolute Percentage Error (MAPE): 2.55%
Median Absolute Percentage Error (MDAPE): 1.81%
```

We have also created the function of making a prediction based on the current data on the ETF price of a future date. The interval between the future date and the latest date from the dataset can be adjusted by changing `pred_int`. Currently, `pred_int` is set to 1 to predict the next day's ETF close price. For example, `pred_int` can be set to 5 to predict the next week's (5 business days hereon) ETF close price. 

```
The close price for ETF on 2022-03-21 is 66.36
The predicted close price for the next day is 66.33999633789062
```

### Flow Chart

The flow chart can be seen below. 

![Machine Learning Flow Chart](../machine_learning_model/Resources/ml_flow_chart1.jpeg)

## Dashboard

We export the data from [`master.ipynb`](../main/master.ipynb) into CSV files, and then import the CSV files into **Tablueau Public** to create interactive dashboard.

Here is the outline of the dashboard:

![Dashboard Blueprint](../presentation/Resources/Dashboard_Outline.png)

The dashboard will include the following viz:
1. A Heatmap for Energy ETF (RYE) portfolio breakdown
2. A bar chart showing the market value of the individual company within RYE
3. A time-series plot showing both the Brent crude oil price and RYE price
4. A time-series plot showing the seasonal changes on oil and RYE price
5. A time-series plot showing the RYE price and trading volume

The interactive elements:
* There will be a linkage between the heatmap and bar chart, so that user can filter the data by **Sector**, and both charts will be updated based on the selected Sector

## Technologies Used

Details can also be found in the [technologies](../technologies) branch. 

### Data Cleaning and Analysis
Pandas will be used to clean the data and perform an exploratory analysis. Further analysis will be completed using Python/Jupyter Notebook/Jupyter Lab.

### Database Storage
PostgreSQL is the database we intend to use, and we will connect to the database through Python/Jupyter Notebook/Jupyter Lab.

### Machine Learning

Regression Analysis, Random Forest, TensorFlow with Keras Sequential Model are the Machine Learning libraries we'll be using. Please refer to the [Machine Learning Models](https://github.com/kobertlam/Oil_Price_and_Stock_Price_Analysis#machine-learning-model) section. Details can also be found in the [machine_learning_model](https://github.com/kobertlam/Oil_Price_and_Stock_Price_Analysis/tree/machine_learning_model) branch. 

### Dashboard
Tableau will be used to create a interactive dashboard. 


