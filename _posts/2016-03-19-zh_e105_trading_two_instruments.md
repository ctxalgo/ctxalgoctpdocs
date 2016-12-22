---
title: 同时交易两个品种
layout: post
category: root
language: zh
---

这个示例展示了如何在一个策略中同时交易两个品种，多个品种的情况类推：

1. 在策略的`__init__`方法的`instrument_ids`参数中指定要交易的所有的品种的ID。
2. 在使用`has_position`，`has_pending_order`和`change_position_to`等API时，通过`instrument_id`参数来指定要操作的品种。

本示例还展示了如何通过设置`backtest`方法中的`profits`和`actual_instrument_ids`参数来在回测的历史数据中获取额外的字段。


```python
import talib
import numpy as np
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *


class TwoInstrumentStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

    def on_bar(self, instrument_id, bars, tick):
        # In a multiple instrument trading strategy, Use the instrument_id parameter in APIs such as
        # has_pending_order, has_position and change_position_to to specify the instrument you want to operate.
        if not self.has_pending_order(instrument_id=instrument_id) and self.in_market_period(instrument_id=instrument_id, delta=timedelta(minutes=20)):
            ohlc = self.ohlc(instrument_id=instrument_id)
            # You can now access to the profits and actual instrument ids.
            profit = ohlc.profits[-1]
            actual_instrument_id = ohlc.actual_instrument_ids[-1]
            if ohlc.length >= self.parameters.slow_ma_period:
                closes = np.array(ohlc.closes)
                ma_fast = talib.SMA(closes, timeperiod=self.parameters.fast_ma_period)
                ma_slow = talib.SMA(closes, timeperiod=self.parameters.slow_ma_period)
                signal = cross_direction(ma_fast, ma_slow)
                if signal != 0:
                    # Change position according to signal.
                    self.change_position_to(signal, instrument_id=instrument_id)


def main():
    start_date = '2015-01-01'
    end_date = '2015-12-31'
    config = {
        'instrument_ids': ['cu00', 'rb00'],  # Specify multiple instrument ids to trade.
        'periods': [Periodicity.FIFTEEN_MINUTE],
        'parameters': {
            'fast_ma_period': 8,
            'slow_ma_period': 15,

        }
    }
    report, data_source = backtest(
        TwoInstrumentStrategy, config, start_date, end_date, profits=True, actual_instrument_ids=True)
    print(report)


if __name__ == '__main__':
    main()

```
