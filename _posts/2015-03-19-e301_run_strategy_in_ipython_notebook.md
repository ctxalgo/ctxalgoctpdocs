---
title: Run strategy in ipython notebook
layout: post
category: ipython
language: zh
---



```python
# Import a trend following strategy.
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
```


```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.

# The values in config will be used to instantiate the strategy objects by the backtest method.
config = {
    'instrument_ids': ['IF99'],                      # We are trading this future instrument.
    'strategy_period': Periodicity.FIFTEEN_MINUTE,   # The ohlc bar granularity on which trading happens.
    'parameters': {
        'fast_ma_period': 8,   # The parameter for the fast moving average.
        'slow_ma_period': 15,  # The parameter for the slow moving average.
    }
}
```


```python
# Backtest the strategy.
report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)
```


```python
# Use charting facility to visualize trades.
c = ChartsWithReport(report, data_source, folder=report.base_folder, open_in_browser=True)
c.set_instrument(config['instrument_ids'][0])
c.period(config['strategy_period'])

c.ohlc()                                        # Draw ohlc bars.
c.volume()                                      # Draw volumes bars.

c.ma(config['parameters']['fast_ma_period'])    # Draw the fast moving average.
c.ma(config['parameters']['slow_ma_period'])    # Draw the slow moving average.

c.drawdowns(max_drawdown_color='pink')          # Visualize the max-drawdown period(s).
c.instrument_positions()                        # Draw the current positions that we are holding.
c.transactions()                                # Draw all the transactions (open and close of a position).
c.balance()                                     # Draw the balance.
c.show()                                        # Open the html page in browser.
```
