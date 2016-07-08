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
from ctxalgoctp.ctp.backtesting_utils import *


class TwoInstrumentStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder, description, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder,
            strategy_period=strategy_period, description=description, logger=logger)

    def on_before_run(self, strategy):
        # Keep track of the number of days we are holding some position using self.context.
        # Information kept in it persists across trading days.
        if 'holding_days' in self.context:
            for sid in self.instrument_ids():
                self.context['holding_days'][sid] += 1
        else:
            self.context['holding_days'] = {sid: 0 for sid in self.instrument_ids()}

    def on_bar(self, instrument_id, bars, tick):
        # In a multiple instrument trading strategy, Use the instrument_id parameter in APIs such as
        # has_pending_order, has_position and change_position_to to specify the instrument you want to operate.
        if not self.has_pending_order(instrument_id=instrument_id):
            # We close a position
            if self.has_position(instrument_id=instrument_id) and self.context['holding_days'][instrument_id] > 3:
                self.change_position_to(0, instrument_id=instrument_id)
            else:
                self.change_position_to(1, instrument_id=instrument_id)
            self.context['holding_days'][instrument_id] = 0


def main():
    start_date = '2015-01-01'
    end_date = '2015-01-31'
    config = {
        'instrument_ids': ['cu99', 'rb99'],  # Specify multiple instrument ids to trade.
        'strategy_period': Periodicity.FIVE_MINUTE,
    }
    backtest(TwoInstrumentStrategy, config, start_date, end_date)


if __name__ == '__main__':
    main()

```
