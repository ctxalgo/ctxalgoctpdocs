---
title: Using the plain backtester
layout: post
category: en
---

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

Then, we implements a simple double moving average trend following strategy using plain backtester.
See [here](e100_trend_following_strategy.html) for the definition of the strategy.
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
backtester = PlainBacktester(initial_capital=1000000.0, has_commission=True, slippage_model=VolumeBasedSlippageModel())
backtester.set_instrument_ids([instrument_id])

# Iterating through the ohlc bars. We start from parameters['slow_ma_bars'], because
# before this bar, there won't be enough data to to calculate slow moving average.
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
```

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
