---
title: Use external data source in backtesting
layout: post
category: root
language: en
---

When you want to conduct multiple backtesting sessions, and each session uses part of a data, you can create
an external data source using the `data_source_for_strategy` method, and pass this data source to your `backtest`
method.

The advantage of this is that you only need to get historical data in one go, which will save you time when you need
to backtest multiple sessions, and each session has slightly different date range.


```python
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import *


def main():
    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['cu00'],
        'periods': [Periodicity.FIFTEEN_MINUTE],
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        }
    }

    # Get a data source containing all the data we need.
    ds = data_source_for_strategy(TrendFollowingStrategy, config, '2014-01-01', '2014-02-28')

    # Then perform two backtests, each one only uses part of the data from the data source.
    backtest(TrendFollowingStrategy, config, '2014-02-01', '2014-02-28', data_source=ds)
    backtest(TrendFollowingStrategy, config, '2014-01-01', '2014-01-31', data_source=ds)


if __name__ == '__main__':
    main()

```
