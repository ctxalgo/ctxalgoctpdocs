---
title: Locating ohlc bars in trading days
layout: post
category: root
language: en
---

This example shows how to locate ohlc bars in trading days. For example, how to get all the bars
from current trading day, how to get bars from yesterday by using the `today_ohlc`, `yesterday_ohlc` and
the more general `ohlc_in_last_days` method.


```python
from ctxalgoctp.ctp.backtesting_utils import *
```

The following listing shows a dummy strategy class, and the `on_bar` method locates ohlc bars in different trading days.

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

Now, we create configurations for backtesting the strategy.

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
