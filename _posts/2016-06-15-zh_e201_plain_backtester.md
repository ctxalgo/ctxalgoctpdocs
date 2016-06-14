---
title: 使用简化回测器
layout: post
category: zh
---

## 1. 简化回测器的介绍
有时候，你只想快速测试一个策略的想法，这时候，你可以使用`PlainBacktester`。这个类提供了一组和`AbstractStrategy`类似
的API，让你能够快速进行策略原型的编程和实验。使用`PlainBacktester`的好处是它比标准的回测器要快很多，原因如下：
1. `PlainBacktester`不进行真实交易时的各种事件回调的仿真，它只接受来自ohlc的K线数据。
2. `PlainBacktester`默认所有的交易都立即全部执行。
本例子展示如何用`PlainBacktester`书写一个双均线趋势跟踪策略。

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

首先，我们使用`get_data_source`来获取所需要的K线数据。

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

## 2. 使用简化回测器来写交易策略
我们来实现一个双均线趋势跟踪策略。策略的逻辑可参见[internal-link](这里 starterkit/e100_trend_following_strategy.py)。
我们实例化一个简化的回测器，指定初始资本100万，并且指定在交易时使用真实的手续费。然后，我们迭代K线数据，并且
使用`change_position_to`方法来改变持仓仓位。

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

## 3. 查看策略回测结果并绘图
在回测完成后，我们可以查看结果。我们先打印出交易报表，然后在浏览器中显示交易细节。

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

## 4. 使用简化回测器书写多品种的交易策略
在此，我们用简化回测器书写一个交易多个品种的策略。与之前的过程一样，我们先构造一个对品种的的数据源`data_source2`，然后
使用`multiple_bars_iterator`来对对个品种的K线进行迭代。当有新的K线的时候，我们调用`update_price`把新的价格信息设置到简化
回测器中。注意，这里使用的`update_price`方法中的`ohlcs`参数来传入多个品种的K线信息。

```python
instrument_ids = ['cu99', 'rb99']
data_source2 = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)

backtester2 = PlainBacktester(instrument_ids=instrument_ids)
for ohlcs in data_source2.multiple_bars_iterator(period=Periodicity.THIRTY_MINUTE, instrument_ids=instrument_ids):
    # Update backtester2 with price from two ohlcs.
    backtester2.update_price(ohlcs=ohlcs)
    # Do actual trading here.




```
