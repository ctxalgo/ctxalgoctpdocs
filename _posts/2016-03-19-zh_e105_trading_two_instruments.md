---
title: 同时交易两个品种
layout: post
category: root
language: zh
---

这个示例展示了如何在一个策略中同时交易两个品种，多个品种的情况类推：

1. 在策略的`__init__`方法的`instrument_ids`参数中指定要交易的所有的品种的ID。
2. 在使用`has_position`，`has_pending_order`和`change_position_to`等API时，通过`instrument_id`参数来指定要操作的品种。


```python
from ctxalgoctp.ctp.backtesting_utils import *


class TwoInstrumentStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

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
        'instrument_ids': ['cu00', 'rb00'],  # Specify multiple instrument ids to trade.
        'periods': [Periodicity.FIVE_MINUTE],
    }
    backtest(TwoInstrumentStrategy, config, start_date, end_date)


if __name__ == '__main__':
    main()

```
