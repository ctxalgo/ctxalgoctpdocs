---
title: 使用外部data source进行回测
layout: post
category: root
language: zh
---

如果你需要进行多次回测，并且每次回测使用的某一连续区间的部分历史数据，你可以使用`data_source_for_strategy`方法先获得所有
历史数据的一个data source，然后将其传递到`backtest`方法中去。这样做的好处是可以节省多次获取外部数据的时间开销。


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
