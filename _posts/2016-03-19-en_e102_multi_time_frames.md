---
title: Multi-time frame strategy
layout: post
category: en
---

This example shows how to register ohlcs for different time periods. You do this by specifying the `periods` parameter
of the strategy's`__init__` method.


```python
from ctxalgoctp.ctp.backtesting_utils import *
```

In the `periods` parameter, we specifies two periods: 15 minutes and 30 minutes. And as before, the `startegy_period`
parameter defines the period on which trade should happen (note we do not trade in this example).

The config object will be used to initialize a strategy object.

```python
config = {
    'instrument_ids': ['IF99'],                      # We are trading this future instrument.
    'strategy_period': Periodicity.FIFTEEN_MINUTE,   # The ohlc bar granularity on which trading happens.
    'periods': [Periodicity.FIFTEEN_MINUTE, Periodicity.THIRTY_MINUTE],
}
```

The following is the strategy code, when one or more ohlc bars arrive, the `on_bar` method will be invoked with
all the bars' information.

```python
class MultiTimeFrameStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

    def on_bar(self, instrument_id, bars, tick):
        # We do not do trading here, instead, we only print out the arrived ohlc bars.
        # Since we only asked for time-based ohlcs, we get them from bars[self.time_based].
        for period, bar in bars[self.time_based].items():
            print('Period={}\tTime={}\tClose price={}'.format(Periodicity.name(period), bar.timestamp, bar.close))
```

Now, we create configurations for backtesting the strategy.

```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.
report, data_source = backtest(MultiTimeFrameStrategy, config, start_date, end_date)

```
