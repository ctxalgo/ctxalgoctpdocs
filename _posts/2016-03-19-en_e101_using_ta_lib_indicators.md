---
title: Using financial indicators from ta-lib in strategies
layout: post
category: root
language: en
---

This example shows how to use indicators from [ta-lib](https://github.com/mrjbq7/ta-lib), a library that
includes hundreds of popular financial indicators, such as SMA, EMA, MACD, RSI.

The full list of ta-lib indicators can be found [here](https://github.com/mrjbq7/ta-lib/tree/master/docs/func_groups).

```python
import numpy as np
import talib
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
        if self.ohlc().length >= self.parameters.slow_ma_period:
            # Construct a numpy array from Python array, as talib requires numpy arrays as inputs,
            # then calculate the simple moving average function SMA.
            closes = np.array(self.ohlc().closes)
            ma_fast = talib.SMA(closes, timeperiod=self.parameters.fast_ma_period)
            ma_slow = talib.SMA(closes, timeperiod=self.parameters.slow_ma_period)

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

report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)

```
