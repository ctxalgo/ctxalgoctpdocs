---
title: Using financial indicators from ta-lib in strategies
layout: post
category: en
---

This example shows how to use indicators from [ta-lib](https://github.com/mrjbq7/ta-lib), a library that
includes hundreds of popular financial indicators, such as SMA, EMA, MACD, RSI.
The full list of ta-lib indicators can be found [here](https://github.com/mrjbq7/ta-lib/tree/master/docs/func_groups).

```python
import talib
import numpy as np
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *
```

The following listing shows the strategy class. Inside the `on_bar` method, we use `talib.SMA` to calculate
simple moving averages.

```python
class TrendFollowingStrategy(AbstractStrategy):
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
        # Here, self.slow_ma_period is a syntax shorthand for self.parameters['slow_ma_period'].
        if self.ohlc().length >= self.slow_ma_period:
            # Construct a numpy array from Python array, as talib requires numpy arrays as inputs,
            # then calculate the simple moving average function SMA.
            closes = np.array(self.ohlc().closes)
            ma_fast = talib.SMA(closes, timeperiod=self.fast_ma_period)
            ma_slow = talib.SMA(closes, timeperiod=self.slow_ma_period)

            # Check if fast moving average up-crosses or down-crosses slow moving average.
            # cross_direction return 1 if ma_fast up-crosses ma_slow, -1 if ma_fast down-crosses ma_fast,
            # and 0 otherwise. The strategy uses this signal to change its position of the traded instrument.
            signal = cross_direction(ma_fast, ma_slow)
            if signal != 0:
                # Change position according to signal.
                self.change_position_to(signal)
```

Now, we create configurations for backtesting the strategy.

```python
from ctxalgoctp.ctp.starterkit.e100_trend_following_strategy import start_date, end_date, config
report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)

```
