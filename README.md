
# Data Collection Layer
- This layer works independently from the rest of the system
- Users do not make direct calls to any functions in this layer through the frontend
- Data is stored and then accessed by the rest of the system.
- Users make modifications to the behaviour of the collection through changing the fields in the configuration file:
--  The interval data is fetched (default: 1 minute for market, 15 minutes for media)
--  The amount of coins sorted by market cap (default: top 50 coins)
--  Filter out unwanted currencies (default: stable coins)
--  Modify reference currency (default: USD)
--  How far back historical data is collected (default: 90 days)
--  to be expanded
- The backend reads the configuration file at every set interval to update the behaviour.

Example ```config.json```: 
```bash
{
  "top-number-of-coins": 50,
  "market-interval": 1,
  "media-interval": 15,
  "currency": "usd",
  "historical-data-days": 90,
  "stable-coin-keywords": ["usd", "usdt", "usdc", "busd", "dai", "tusd", "usdp", "usdd", "gusd", "fdusd"],
  "coins_ignored": []
}
```
## Implemented APIs
Currently CoinGecko, CryptoCompare and Reddit API are integrated.

### CoinGecko 
---
For live data, CoinGecko was chosen to fetch top 50 currencies:
| API               | Free Access | Market Cap Sorting | Pagination | Rate Limits | API Key Needed? | Reliability | Notes                                  |
|-------------------|-------------|---------------------|------------|-------------|------------------|-------------|----------------------------------------|
| **CoinGecko**     | ✅ Yes      | ✅ Yes              | ✅ Yes      | Generous    | ❌ No             | ✅ High      | Full data, simple access               |
| **CoinMarketCap** | ✅ (limited)| ✅ Yes              | ✅ Yes      | Strict      | ✅ Yes            | ✅ High      | Key required, some quotas              |
| **CryptoCompare** | ✅ Yes      | ❌ No (raw list)    | ❌ No       | Moderate    | ✅ Yes            | ✅ High      | No built-in sorting                    |
| **Binance API**   | ✅ Yes      | ❌ No (by pair only)| ❌ No       | Good        | ❌ No             | ✅ High      | Only exchange-specific data            |

Data stored from CoinGecko: 
- Timestamp (normalised to UTC) in HH:MM:SS.XXX
- Symbol
- Price
- Market cap
- Total volume
- 24hr price change %
- 24hr market change %

### CryptoCompare
---
Historical data collection

| API               | Free Access | Historical OHLCV | Granularity         | Backfill Range     | API Key Needed? | Rate Limits | Reliability | Notes                                       |
|-------------------|-------------|------------------|----------------------|---------------------|------------------|-------------|-------------|---------------------------------------------|
| **CryptoCompare** | ✅ Yes      | ✅ Yes           | Minute, Hour, Day   | Years (depends)     | ✅ Yes (free)    | Moderate    | ✅ High      | Best historical depth, includes volume      |
| **CoinGecko**     | ✅ Yes      | ✅ Yes (limited) | Daily only          | ~90 days max        | ❌ No            | Generous    | ✅ High      | Good for spot-checks, not deep history      |
| **CoinMarketCap** | ✅ (limited)| ✅ Yes           | Minute, Hour, Day   | Depends on tier     | ✅ Yes           | Strict      | ✅ High      | Requires paid tier for full range           |
| **Binance API**   | ✅ Yes      | ✅ Yes           | 1m to 1M (monthly)  | Exchange-specific   | ❌ No            | Good        | ✅ High      | Only includes Binance-listed assets         |

Data stored from Crypto Compare: 
- Timestamp (normalised to UTC) in YYYY:MM:DD
- Open (the price at the start of the day)
- High (highest price during the day)
- Low (lowest price during the day)
- Close (the price at the end of the day)
- Volume (of the day)

### Issue with CryptoCompare
Data is outdated, for example: 
![No update for 4 months](https://cdn.discordapp.com/attachments/1387074882268958831/1389805877208416256/image.png?ex=6865f516&is=6864a396&hm=b8873d2a5e75efde64a7d13c7656d7cb361e2d0024b31fc8282363a393b01ab4&)
![enter image description here](https://cdn.discordapp.com/attachments/1387074882268958831/1389806220189237372/image.png?ex=6865f568&is=6864a3e8&hm=51b5e52eae04655517a0ca87658610deeebf90b46fb05f1c9a425c01853c63c1&)

Solution to this issue: Implement backup APIs to fetch data from if the primary API has failed doing so. 

## Data Storage
Logs are appended to files in the `logs/` folder with entries like:

Historical data: ``
{symbol}.csv
`` (example ``BTC.csv``)
| Date       | Open     | High     | Low      | Close    | Volume         |
|------------|----------|----------|----------|----------|----------------|
| 2025-03-30 | 82625.65 | 83516.72 | 81563.47 | 82379.51 | 839590269.08   |
| 2025-03-31 | 82379.51 | 83917.40 | 81290.13 | 82539.52 | 1894666416.64  |
| 2025-04-01 | 82539.52 | 85554.98 | 82419.98 | 85174.96 | 1962356062.59  |
| 2025-04-02 | 85174.96 | 88505.24 | 82299.20 | 82490.58 | 3519914220.40  |
| 2025-04-03 | 82490.58 | 83928.41 | 81178.57 | 83158.80 | 2335462308.44  |
| 2025-04-04 | 83158.80 | 84716.02 | 81648.68 | 83860.21 | 3085885741.07  |
| 2025-04-05 | 83860.21 | 84230.44 | 82357.52 | 83503.37 | 622496989.40   |

Live data: ``
{api}.csv
`` (example ``coingecko.csv``)
| Timestamp     | Symbol | Price     | Market Cap     | Total Volume   | 24h Price Change (%) | 24h Market Cap Change (%) |
|---------------|--------|-----------|----------------|----------------|------------------------|----------------------------|
| 12:35:02.229  | btc    | 107190    | 2131455651436  | 31514043904    | 0.12328                | 0.09653                    |
| 12:35:02.229  | eth    | 2443.93   | 294888666880   | 18149262316    | 0.99369                | 0.82592                    |
| 12:35:02.229  | xrp    | 2.17      | 127796191896   | 2225314671     | -1.09381               | -1.02644                   |
| 12:35:02.229  | bnb    | 643.22    | 93829511886    | 659794268      | -0.58924               | -0.6376                    |
| 12:35:02.229  | sol    | 143.38    | 76609520799    | 3802918286     | -1.60658               | -1.06942                   |

### Reddit API: 
---
- Limited subreddits dedicated for the top coins
- Limited valuable information
- Requires extensive amount of quality filtering through posts

Media data: ``
{api}_{symbol}_{date}.txt
`` (example ``Reddit_BTC_2025-06-28.txt``)
```
Time Posted: 2025-06-30 16:21:35
Title: Physical Bitcoin Gift's I have been giving as presents the past 4 years.
Text: For the past four years, instead of giving $50 gifts that often end up unused or tossed aside for birthdays and Christmas, I’ve been making custom physical bitcoins as gifts instead. I load each coin with about $50 worth of my local currency (AUD), keeping the Satoshi amounts rounded for simplicity, like 0.0025, 0.0015, 0.001, and now 0.0005 BTC.  Over the years, this little tradition has definitely opened a lot of eyes among my friends and family and has even orange pilled quite a few of them!
Upvotes: 154
Upvote Ratio: 0.97
Flair: None

Time Posted: 2025-06-30 14:21:45
Title: 🚨 JUST IN: Metaplanet bought yet another 1,005 $BTC worth $108.1M, bringing its total holdings to 13,350 $BTC in its treasury.
Text: 
Upvotes: 147
Upvote Ratio: 0.97
Flair: None

Time Posted: 2025-06-30 12:12:28
Title: 1 BTC = 1 BTC
Text: 
Upvotes: 149
Upvote Ratio: 0.91
Flair: None

Time Posted: 2025-06-30 10:29:44
Title: Your watching bitcoin aren't you?
Text: As of the past couple of days bitcoin seems to be holding the 106k lvl,   It also seems we were are making bullish structure on smaller timeframes.  so based off this daily chart what is you opinion of direction? 
Upvotes: 206
Upvote Ratio: 0.86
Flair: None

Time Posted: 2025-06-30 09:54:40
Title: Uh oh, btc death spiral confirmed, time to sell boys
Text: Cathie Wood is notoriously good at making incorrect predictions. If shes claiming btc will moon then its for sure going to dip. Sell now and start restacking after she sells hers for a loss. She is the anti-prophet. You've been warned. 
Upvotes: 186
Upvote Ratio: 0.72
Flair: None
```
## Code Demonstration
The config file was modified to the following for the demonstration:
```bash
{
  "top-number-of-coins": 5,
  "market-interval": 10, #seconds
  "media-interval": 20, #seconds
  "currency": "usd",
  "historical-data-days": 90,
  "stable-coin-keywords": ["usd", "usdt", "usdc", "busd", "dai", "tusd", "usdp", "usdd", "gusd", "fdusd"],
  "coins_ignored": []
}
```
### Major functions: 
``getTopCoins``: The code fetches market data from CoinGecko for the top N cryptocurrencies, filters out stablecoins and ignored coins, and returns the top symbols after filtering.
```c
Function isStableCoin(coin, keywords):
    If coin name or symbol contains any keyword → it's stable
    If coin price is around $1 → it's stable
    Return True if either is true

Function getTopCoins(how_many, search_limit, currency, config):
    Get top 'search_limit' coins from CoinGecko

    Get list of stable keywords and ignored coins from config

    Remove stablecoins and ignored coins from the list

    Keep the first 'how_many' coins from the result

    Log and return their symbols in uppercase
```

``collectHistoricalData``: loops through a list of coins, fetches their historical data, logs it, and handles any errors individually.
```c
Function fetchDailyHistory(symbol, currency, days):
    Check if symbol needs to be overridden → update symbol

    Set CryptoCompare daily history API URL
    Set parameters: from symbol, to currency, and number of days

    Send GET request with these parameters
    If it fails, raise an error
    Parse the JSON response

    Return the list of daily historical data
   
Function collectHistoricalData(symbols, currency, days):
    For each symbol in the list:
        Try:
            Print that it's being fetched
            Get historical data for that symbol
            Log the data
            Wait 0.5 seconds (to avoid rate limits)
        If there's an error:
            Print the error for that symbol
```
``continuousCollection``: runs in a loop to fetch and log top cryptocurrency data daily, detect new coins, and periodically collect related media, while handling errors and avoiding duplicates.
```c
Function continuousCollection():
    Load config from file
    Set up tracking variables (last_top_symbols, last_run_day, minute_count)

    Loop forever:
        Try:
            Reload config
            Read top coin settings, currency, and days

            Get current time and day
            Calculate how far into the day we are

            If it's the end of the day and hasn't run yet:
                Print daily update message
                Get top coins
                Collect full historical data
                Save the top coins and mark today as run
                Collect media
                Reset minute counter

            Else:
                Get top coins again
                Find any new coins not seen before

                If there are new coins:
                    Print them
                    Fetch their historical data
                    Update known coins

                Else:
                    Just print a message (no change)

                Increase minute counter
                If time for media:
                    Collect media
                    Reset counter

        If there's an error:
            Print it

        Wait {interval} seconds before repeating
```


## Extending
- Extend to store data in databases (e.g., SQLite, Postgres).
- Live data could be summarised each day and later be stored as historical data so more in-depth analysis can be performed.
- Expand for text data.
- Remove live data that is more than 24hrs old to limit file size (this is much easier to complete in SQL).
- Implement backup APIs.
