---
title: 如何定位不同交易日中的K线
layout: post
category: zh
---

本示例展示如何定位不同交易日中的K线，比如如何获得当前交易日的所有K线，如何获得昨天的所有K线。通过使用`today_ohlc`方法，
`yesterday_ohlc`方法，以及更基础的`ohlc_in_last_days`方法可以很方便的实现这些功能。

```python
from ctxalgoctp.ctp.backtesting_utils import *
```

以下代码段展示了一个简单的交易策略，在`on_bar`方法中，我们定位来自不同交易日的K线。

```python
class DummyStrategy(AbstractStrategy):
    """
    A single double moving average trend following strategy:
    1. If the fast moving average up-cross the slow moving average, change position to 1.
    2. If the fast moving average down-cross the slow moving average, change position to -1.
    That is, the position can be 0, -1 and 1.
    """
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

    def on_bar(self, instrument_id, bars, tick):
        ohlc = self.ohlc()
        today_ohlc = self.today_ohlc()
        print('Today bars:\t{}\tto\t{}'.format(today_ohlc.dates[0], today_ohlc.dates[-1]))

        # Locating the ohlc for yesterday. In the first day of backtesting,
        # we won't have any bar from yesterday, so we need to check if the result
        # from yesterday_ohlc is None or not.
        yesterday_ohlc = self.yesterday_ohlc()
        if yesterday_ohlc is not None:
            print('Yesterday bars:\t{}\tto\t{}'.format(yesterday_ohlc.dates[0], yesterday_ohlc.dates[-1]))

        # # We can ask for ohlc for any day range using bars_in_last_days. Specify start and end
        # # using the same values as you slice a Python array backwards. The difference here
        # # is that the slice unit is in days.
        # # Here we ask for bars from the fourth and third to the last trading days.
        ohlc_in_last_days = self.ohlc_in_last_days(start=-4, end=-2)
        if ohlc_in_last_days is not None:
            print('Fourth and third to the last day bars:\t{}\tto\t{}'.format(
                ohlc_in_last_days.dates[0], ohlc_in_last_days.dates[-1]))

```

现在我们对以上交易策略进行历史数据的回测。

```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.

# The values in config will be used to instantiate the strategy objects by the backtest method.
config = {
    'instrument_ids': ['IF99'],                      # We are trading this future instrument.
    'strategy_period': Periodicity.FIFTEEN_MINUTE,   # The ohlc bar granularity on which trading happens.
}

report, data_source = backtest(DummyStrategy, config, start_date, end_date)

```
