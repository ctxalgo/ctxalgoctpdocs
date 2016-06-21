---
title: Using the plain backtester
layout: post
category: en
---


## 1. Introduction to plain backtester

When you want to do a fast prototype of a strategy just be iterating over ohlcs, you can use the `PlainBacktester`
class. This class provides a API set similar to that of `AbstractStrategy`, which is what you use when writing
real strategies. The benefit of using `PlainBacktester` is that it runs much faster than the actual backtester.
`PlainBacktester` runs faster because:

1. It does not simulate trading events, it only accepts close price from ohlc bars.
2. It assumes all trades are executed immediately on bar close.

This example demonstrates `PlainBacktester` with a double moving average trend following strategy.


```python
import os
import talib
import numpy as np
from datetime import datetime

from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.ta.cross import cross_direction
from ctxalgolib.charting.charts import ChartsWithReport

from ctxalgoctp.ctp.backtesting_utils import get_data_source
from ctxalgoctp.ctp.plain_backtester import PlainBacktester
from ctxalgoctp.ctp.slippage_models import VolumeBasedSlippageModel
```

First, we get a data source for some instrument to trade by using the `get_data_source` method.
From the retrieved data source, we can get desired ohlcs.

```python
instrument_id = 'cu99'
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.
base_folder = os.path.join(os.environ['CTXALGO_TEST'], 'strategies', 'plain_backtester')
data_period = Periodicity.HOURLY
data_source = get_data_source([instrument_id], base_folder, start_date, end_date, data_period)

# Get ohlc from data source. The ohlc object contains the required data for the instrument.
ohlc = data_source.ohlcs()['time-based'][instrument_id][data_period]
```


## 2. Use plain backtester to write a trading strategy

Here, we implements a simple double moving average trend following strategy using plain backtester.
See [here]({% post_url 2016-03-19-en_e100_trend_following_strategy %}) for the definition of the strategy.

We create a plain backtester and then iterating the bars from the retrieved ohlc data. During the iteration,
we call `change_position_to` method from the plain backtester to open and close positions.


```python
parameters = {
    'slow_ma_bars': 15,
    'fast_ma_bars': 8,
}

# Use talib to calculate the two moving averages. Because we already know all the ohlc bars,
# we can calculate the whole fast and slow moving averages in one go.
fast_ma = talib.SMA(np.array(ohlc.closes), timeperiod=parameters['fast_ma_bars'])
slow_ma = talib.SMA(np.array(ohlc.closes), timeperiod=parameters['slow_ma_bars'])

# Create a plain backtester, set the initial capital to one million,
# You can set the initial capital, and you can decide if the backtester will include trade commission or slippage.
# Both commission and slippage contribute to transaction cost, which will affect your backtesting result.
# Here we set the backtester to include commission and use `VolumeBasedSlippageModel` to introduce slippage.
# Other slippage models are available at `ctxalgoctp.ctp.slippage_models'. If you set `slippage_model` to None,
# No slippage will be included.
backtester = PlainBacktester(
    initial_capital=1000000.0,                  # Initial capital
    has_commission=True,                        # Include commission in trading
    slippage_model=VolumeBasedSlippageModel())  # Setup slippage model
backtester.set_instrument_ids([instrument_id])

# Iterating through the ohlc bars. We start from parameters['slow_ma_bars'], because
# before this bar, there won't be enough data to to calculate slow moving average.
start_time = datetime.now()
for bar in range(parameters['slow_ma_bars'], ohlc.length):
    price = ohlc.closes[bar]
    timestamp = ohlc.dates[bar]
    volume = ohlc.volumes[bar]

    # We update the current price from the bar close price into the plain backtester. This is important.
    # Only with the updated instrument price, the backtester can correctly calculate balance, profit.
    # It is recommended to do price update as soon as you have the newest price, so the backtester
    # can calculate balance, profit more accurately.
    backtester.update_price(price, volume, timestamp, instrument_id=instrument_id)

    # Check if the fast moving average up or down crosses the slow moving average.
    # Change our position according to this cross information.
    signal = cross_direction(fast_ma, slow_ma, offset=-bar)
    if signal != 0:
        backtester.change_position_to(price, signal, timestamp, instrument_id=instrument_id)
end_time = datetime.now()
print('Backtesting duration: ' + str(end_time - start_time))
```

## 3. Investigating backtesting result and draw charts

After backtesting, we can investigate the results. We first print out the backtesting summary, and then chart the
trade details in browser.

```python
report = backtester.report()
print(report)

# Calculate the average number of bars for transactions.
transaction_lengths = [
    t.length_by_bar() for tran_list in report.get_transactions().values()
    for t in tran_list if t.length_by_bar() is not None]
print('Average transaction length by bar: {}'.format(sum(transaction_lengths) / len(transaction_lengths)))

# Chart the ohlc and transactions.
c = ChartsWithReport(data_feed=data_source, report=report)
c.set_instrument(instrument_id)
c.period(data_period)

# Chart ohlc bars and volume.
c.ohlc()
c.volume()

# Draw the fast and slow moving averages.
c.ma(parameters['slow_ma_bars'])
c.ma(parameters['fast_ma_bars'])

c.balance()
c.drawdowns(max_drawdown_color='pink', only_max=False)
c.instrument_positions()
c.transactions(pnl=True)

# Display the chart in browser.
c.show()
```

## 4. Backtesting strategy with multiple instruments

Here, we write a strategy that trades multiple instruments using plain backtester. As usual, we create a data source
with multiple instruments, and use the `multiple_bars_iterator` to iterate over multiple ohlcs. When we have new data
points, we need to use `update_price` to set the price information into the plain backtester. Note here we specify
the `ohlcs` parameter of the `update_price` method to set price for multiple instruments in one statement.


```python
instrument_ids = ['cu99', 'rb99']
data_source2 = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)

backtester2 = PlainBacktester(instrument_ids=instrument_ids)
for ohlcs in data_source2.multiple_bars_iterator(period=Periodicity.THIRTY_MINUTE, instrument_ids=instrument_ids):
    # Update backtester2 with price from two ohlcs.
    backtester2.update_price(ohlcs=ohlcs)
    # Do actual trading here.
```

## 5. Use external data source in plain backtesting

If you have your own ohlc data stored in csv files, you can config the `get_data_source` method to load those files.
The following code listing showing how to do this.


```python
def parse_ohlc_line(line):
    """
    Parse a line from ohlc file into data.
    :param line: string, a line from the ohlc file.
    :return: (timestamp, open, high, low, close_, volume, amount)
        The return is a tuple containing the parsed data:
            timestamp: datetime, the timestamp of the ohlc line.
            open: float, the open price,
            high: float, the high price,
            low: float, the low price,
            close: float, the close price,
            volume: int, the volume,
            amount: float, the amount of money that has been traded, if there is no amount data
                in the raw text, set this to 0.
    """
    parts = line.split(',')
    ts = datetime.strptime(parts[1], '%Y-%m-%d %H:%M:%S')
    open_ = float(parts[4])
    high = float(parts[5])
    low = float(parts[6])
    close_ = float(parts[7])
    volume = float(parts[8])
    return ts, open_, high, low, close_, volume, 0

instrument_ids = ['IF99']
data_paths = {
    'IF99': os.path.join('..', '..', 'tests', 'data', 'IF99_15m_20140101_20141231.csv')
}

data_source3 = get_data_source(
    instrument_ids, base_folder, start_date, end_date, data_period,
    data_paths=data_paths,            # Point data file locations to data source.
    line_parser=parse_ohlc_line)      # Specify how to parse your data files.

backtester3 = PlainBacktester(instrument_ids=instrument_ids)
for ohlcs in data_source3.multiple_bars_iterator(period=Periodicity.FIVE_MINUTE, instrument_ids=instrument_ids):
    # Update backtester3 with price from two ohlcs.
    backtester3.update_price(ohlcs=ohlcs)
    # Do actual trading here.

```
