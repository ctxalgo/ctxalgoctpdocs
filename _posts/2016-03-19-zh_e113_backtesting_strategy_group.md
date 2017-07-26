---
title: Backtesting strategy group
layout: post
category: root
language: zh
---

```python
import talib
import numpy as np
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgoctp.ctp.live_strategy_utils import get_command_line_parser


class TrendFollowingStrategy(AbstractStrategy):
    def on_before_run(self, strategy):
        pass

    def on_bar(self, instrument_id, bars, tick):
        pass


def main():
    start_date = '2016-01-01'  # Backtesting start date.
    end_date = '2016-01-31'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['cu00', 'rb00'],                      # We are trading this future instrument.
        'periods': [Periodicity.FIFTEEN_MINUTE],
        'parameters': {
            'Y': 0.01
        }
    }

    config1 = copy(config)
    # config1['description'] = {'product': 'test', 'name': 's1'}
    config2 = copy(config)
    # config2['description'] = {'product': 'test', 'name': 's2'}
    cmd1 = '--name test.s1 --ohlc-fields book1,profits,actual_instrument_ids'
    cmd2 = '--name test.s2 --ohlc-fields book1,profits,actual_instrument_ids'
    #
    run_backtest([TrendFollowingStrategy]*2, [config1, config2], [cmd1, cmd2], start_date, end_date)


if __name__ == '__main__':
    main()

```
