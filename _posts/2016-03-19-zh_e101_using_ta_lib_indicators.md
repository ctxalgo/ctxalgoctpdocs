---
title: 在策略中使用来自ta-lib的指标
layout: post
category: root
language: zh
---

本示例展示如何在策略中使用来自[ta-lib](https://github.com/mrjbq7/ta-lib)的指标。ta-lib中包括了上百种常用的金融指标,
比如EMA, EMA, MACD, RSI。

[这里](https://github.com/mrjbq7/ta-lib/tree/master/docs/func_groups)包括了ta-lib中的所有指标。


```python
import numpy as np
import talib
from ctxalgolib.ta.cross import cross_direction

from ctxalgoctp.ctp.backtesting_utils import *
```

一下代码段展示了完整的交易策略。在`on_bar`方法中，我们使用`talib.SMA`来计算移动平均线。


```python
class TrendFollowingStrategy(AbstractStrategy):
    """
    A single double moving average trend following strategy:
    1. If the fast moving average up-cross the slow moving average, change position to 1.
    2. If the fast moving average down-cross the slow moving average, change position to -1.
    That is, the position can be 0, -1 and 1.
    """
    def __init__(self, instrument_ids, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

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

现在我们对以上交易策略进行历史数据的回测。


```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.

# The values in config will be used to instantiate the strategy objects by the backtest method.
config = {
    'instrument_ids': ['IF00'],                      # We are trading this future instrument.
    'periods': [Periodicity.FIFTEEN_MINUTE],   # The ohlc bar granularity on which trading happens.
    'parameters': {
        'fast_ma_period': 8,   # The parameter for the fast moving average.
        'slow_ma_period': 15,  # The parameter for the slow moving average.
    }
}

report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)

```
