---
title: Trading two instruments
layout: post
category: root
language: en
---

This example shows how to write a strategy to trade two instruments, and the same goes for more instruments:

 1. Specify the instrument ids that you want to trade in the `instrument_ids` parameter in the `__init__` method of the strategy.
 2. When using APIs such as `has_position`, `has_pending_order` and `change_position_to`, you need to specify the instrument_id for the instrument that you want to operate.


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
    report, data_source = backtest(TwoInstrumentStrategy, config, start_date, end_date)
    print(report)


if __name__ == '__main__':
    main()

```
