---
layout: post
title: "Finana Project Design"
date: 2021-09-27 01:37 -0700
tags: [technology]
---

# Background

After working for several years, I finally save up some money for stock investment. But I don't want to be a blind follower, so I researched and found something called technical analysis. These ideas reminded me of my dad's old stock analysis guide books. He started to study those books when I was a high school student, but I haven't seen him getting rich yet. Therefore I am a little skeptical and would like to experiment out with real data. Also I think it is good chance for me to practice with Python and Panda DataFrame. So, here is this project Finana (**Fin**ance **Ana**lysis) to help with data collection and processing.

# Requirements

Create a web based application to help on stock investment decision for US markets with basic technical analysis. Major functionalities include: create daily report with indicator status of selected stocks for buy/sell/hold decision, and do daily screening on a wide range of stocks for buy/short choices. While screening, also create rankings by market capacity bracket (big, middle, small, tiny).

### Report for Selected Stocks

1. For simplicity, hardcode the lists of stocks, and put them directly in the storage. Input UI is not required, but we need to support multiple lists, e.g. holdings and watch list.

2. Multiple user access and authentication is not required. Just assume anonymous login to all public lists.

3. A backend job should run daily (market open day) to pull the daily price/volume history (quote history) of the stocks in this lists, and calculate the indicators. After that, compare current results with previous ones to show changes if any.

4. Indicators to support:

    Name         | Buy/Hold Signal    | Sell Signal
    ------------ | ------------------ | -----------------
    SMA          | SMA50 > SMA200     | SMA50 <= SMA200
    MACD         | Value > Signal     | Value <= Signal
    RSI          | Value < 70         | Value >= 70
    Supertrend   | Value < Close      | Value >= Close

### Stock Screening

1. Retrieve all stocks in US market, including Dow Jones, Nasdaq, and AMEX once every other week.

2. Indicators calculation is the same as above. But we should use stricter criteria, e.g. the signal line crossing needs to happen within 2 days, in order to keep the result list reasonably short:

    Name         | Trigger                      | Lookback
    ------------ | ---------------------------- | -----------------
    MACD         | Value > Signal and Value < 0 | 2 days
    RSI          | Two dips: 1st < 40, 2nd < 30 | 7 days
    Supertrend   | Value < Close                | 2 days
    Ranking      | Score increases 10% or more  | Current

### Stock Rankings

1. Generate a score for each stock from above by [StockCharts Technical Rank (SCTR)](https://www.investopedia.com/terms/s/stockcharts-technical-rank-sctr.asp), group them by their market capacity, and then sort them by their score within each group.

2. Compare score and ranking with previous report to show changes if any.

# Implementation

System will be implemented with Python 3.8, Flask, Panda Dataframe, and Azure blobs for data persistence. It will be deployed as Azure AppService with WebJob as scheduled backend jobs.

### Fetching Data

I prefer complimentary data source with web API or package. But things like this need some research work. And they may change without notification. Thus I would need consistently monitoring on availability.

1. For getting a complete stock list in US exchanges, I found the download API from [the Nasdaq screener](https://www.nasdaq.com/market-activity/stocks/screener?render=download):

    ```
    GET https://api.nasdaq.com/api/screener/stocks?tableonly=true&limit=25&offset=0&download=true
    ```

2. For quote history, Nasdaq.com has another public web API as below. However, as the query load is big, I need to be cautious with throttling to ensure not exceeding any limit. Also, I have another option, [the yfinance package](https://github.com/zujiancai/yfinance) as backup.

    ```
    GET https://api.nasdaq.com/api/quote/MSFT/historical?assetclass=stocks&fromdate=2020-10-05&limit=9999&todate=2021-10-05
    ```

### Technical Analysis

I don't want to reinvent the wheel, and would use [this handy code](https://github.com/arkochhar/Technical-Indicators) for all indicators.

### Data Persistence

I don't want to use database like SQL or CosmosDb. They are either hard to set up or too expensive. As each type of jobs would run as singleton, I don't need to support concurrent writes. And the data can be easily modelled as files. So Azure blob should be a good fit. Below are the files that will be used. All files should be pickled before uploading to Azure.

File Name             | Collection         | Quantity            | Content
--------------------- | ------------------ | ------------------- | ------------------
StockList             | SourceData         | 1                   | Full list of stocks in US exchanges
SelectLists           | SourceData         | 1                   | Stock lists pre-configured
{Ticker}              | SourceData         | 1 per ticker, ~8K   | History quotes for each stock
{yyyy-MM-dd}_Select   | DailyReport        | 1 per day, keep 10  | Daily report for customized lists
{yyyy-MM-dd}_Screen   | DailyReport        | 1 per day, keep 10  | Daily report for screening results
{yyyy-MM-dd}_Ranking  | DailyReport        | 1 per day, keep 10  | Daily rankings by market cap bracket
FetchListLog          | Control            | 1                   | Lock file and keep track of what day has been triggered for fetching all stocks
FetchQuotesLog        | Control            | 1                   | Lock file and keep track of what day has been triggered for updating historical quotes
{yyyy-MM-dd}_listjob  | Control            | 1 per day, keep 10  | Job status for fetching stock list
{yyyy-MM-dd}_quotejob | Control            | 1 per day, keep 10  | Job status for fetching history. It also serves as a continuation token for each batch

1. StockList and {Ticker} files are similar with a date and the [DataFrame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html) object which contains the raw data:

    ```
    {
        "date": "2021-09-10",
        "content": <DataFrame object>
    }
    ```

    StockList data frame:

    symbol | name                    | market_cap    | contry        | ipo_year | sector     | industry
    ------ | ----------------------- | ------------- | ------------- | -------- | ---------- | -----------------------
    AAPL   | Apple Inc. Common Stock | 2446472047400 | United States | 1980     | Technology | Computer Manufacturing

    {Ticker} data frame:

    date       | open    | high    | low     | close   | volume
    ---------- | ------- | ------- | ------- | ------- | -------- 
    2021-09-10 | 6186.30 | 6188.55 | 6130.25 | 6135.85 | 190400  

2. SelectLists is a collection of pre-configured stock lists for making {yyyy-MM-dd}_Select report:

    ```
    [
        {
            "name": "My Holdings",
            "content": [ "BAC", "MSFT", "NVDA", ... ]
        },
        {
            "name": "Watchlist",
            "content": [ "GOOG", "AMZN", "FB", ... ]
        },
        // ...
    ]
    ```

3. {yyyy-MM-dd}_Select report is for the selected stock lists (each list's result is represented with one data frame):

    ```
    [
        {
            "name": "My Holdings",
            "content": <DataFrame object>
        },
        {
            "name": "Watch List",
            "content": <DataFrame object>
        },
        // ...
    ]
    ```

    Sample data frame:

    symbol | date        | close     | volume       | SMA50     | SMA200     | MACD       | MACD_signal  | RSI        | Supertrend 
    ------ | ----------- | --------- | ------------ | --------- | ---------- | ---------- | ------------ | ---------- | --------------
    NVDA   | 2021-09-30  | 207.16    | 22101000     | 210.53    | 165.82     | -0.77      | 1.7          | 41.83      | 227.35

4. {yyyy-MM-dd}_Screen report only contain the symbols which meet indicator criteria. Each symbol will come with the last data point date and the market cap tier:

    ```
    {
        "MACD": <DataFrame object>,
        "RSI": <DataFrame object>,
        "Supertrend": <DataFrame object>
    }
    ```

    Sample data frame:

    symbol | date        | market_cap    
    ------ | ----------- | -------------
    NVDA   | 2021-09-30  | big

5. {yyyy-MM-dd}_Ranking report:

    ```
    {
        "MegaCap": <DataFrame object>,
        "BigCap": <DataFrame object>,
        "MidCap": <DataFrame object>,
        "SmallCap": <DataFrame object>,
        "TinyCap": <DataFrame object>
    }
    ```

    Sample data frame:

    symbol | date        | score       | rank
    ------ | ----------- | ----------- | ----------- 
    GOOG   | 2021-09-30  | 322.13      | 28

6. FetchListLog and FetchQuotesLog are in same format, and contain the dates that have been triggered:

    ```
    [
        "2021-09-30",
        "2021-09-29",
        "2021-09-28",
        "2021-09-27",
        "2021-09-24",
        // ...
    ]
    ```

7. {yyyy-MM-dd}_listjob contains job status for loading complete stock list:

    ```
    {
        "start_time": "2021-09-30 00:12:32Z",
        "end_time": "2021-09-30 00:13:58Z",
        "status": "success",
        "errors": [],
        "count": 8830
    }
    ```

8. {yyyy-MM-dd}_quotejob contains job status for loading quote history for all stocks. As there are so many data, the work will be divide into many batch, and this file also serves as a continuation token:

    ```
    {
        "start_time": "2021-09-30 00:06:12Z",
        "end_time": "",
        "status": "running",
        "errors": [],
        "completed": [ "ABC", "ABD", ... ],
        "failed": [ "B1K", "DLE", ... ],
        "workqueue": [ "MNO", "MNK", ... ]
    }
    ```

### Backend Jobs

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>
    mermaid.initialize({ startOnLoad: true });
</script>

1. FetchList job to refresh the stock lists of all US exchanges. After getting the stock list from remote, convert it to DataFrame and save it to Azure blob.

<div class="mermaid">
    graph LR
    A[Start] --> B{Aquire Lock on Log}
    B -->|Success| C{Check Last Date}
    B -->|Failure| D[End as No-op]
    C -->|Meet Schedule| E[Create Job Status]
    C -->|Not Meet| D
    E --> F[Load List from Remote and Save]
    F --> G[Log Current Date and Update Status]
    G --> H[Release Lock]
    H --> I[End]
</div>

2. TriggerSequence job to initialize this sequence: fetch historical quotes for all stocks (FetchQuotes), generate select report (ReportSelect), generate screen report and create rankings (ScreenAll). This initializer only check schedule and add the first task in queue.

<div class="mermaid">
    graph LR
    A[Start] --> B{Aquire Lock on Log}
    B -->|Success| C{Check Date}
    B -->|Failure| D[End as No-op]
    C -->|Meet Schedule| E[Add FetchQuotes Task in Queue]
    C -->|Not Meet| D
    E --> F[Log Current Date]
    F --> G[Release Lock]
    G --> H[End]
</div>

3. HandleQuotes to work on the quote related tasks in queue. It can handle different types of task in the sequence, and add new task in queue until the sequence is completed. Along the way, it also update the job status to reflect the sequence progress. 

Below is the task transition. FetchQuotes only fetch a batch of 200 stocks each time and continue until all stocks have been udpated.

<div class="mermaid">
    stateDiagram-v2
    [*] --> FetchQuotes
    FetchQuotes --> FetchQuotes
    FetchQuotes --> ReportSelect
    ReportSelect --> ScreenAll
    ScreenAll --> [*]
</div>

HandleQuotes job logic is like:

<div class="mermaid">
    graph LR
    A[Listen on Task Queue] --> B{Check Task Type}
    B -->|FetchQuotes| C[Fetch Quotes for Batch]
    B -->|ReportSelect| D[Create Select Report]
    B -->|SceenAll| E[Create Screen and Rankings]
    C --> F{More Batch}
    F --> |Yes| G[Add FetchQuotes Task]
    F --> |No| H[Add ReportSelect Task]
    D --> I[Add SceenAll Task]
    E --> J[End]
</div>

4. The web job schedule is more frequent than our expected job frequency in case of any failure. Therefore, the job may retry a couple of times, need check the schedule or existing log in code.

    Job Type         | Schedule          | Lock File        | Description
    ---------------- | ----------------- | ---------------- | -----------------------------------
    FetchList        | 0 1 0 */2 * *     | FetchListLog     | Load stock list for all US exchanges
    TriggerSequence  | 0 0 18-23 * * 1-5 | FetchQuotesLog   | Trigger sequence for historical quotes handling
    HandleQuotes     | 0 */5 * * * *     | Azure Queue      | Handler sequence tasks in queue

### Frontend Web UI

### Deployment

### Monitoring and Trouble-Shooting


# Revision

Version      | Start Date        | Description
------------ | ----------------- | -----------------------
1.0          | 2021-09-27        | Created