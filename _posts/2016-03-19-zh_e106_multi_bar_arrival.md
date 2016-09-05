---
title: 多K线同时递增时的回调函数
layout: post
category: root
language: zh
---

在同时交易多个品种的策略中，有时候我们希望在所有品种都递增了一根K线时，获得一个函数回调。比如，在配对交易中，你希望
这两个交易的品种都增加了一根K线的时候，重新评估对冲获利空间。这时，你可以使用`add_bar_arrival_action`来达到此目的。

`add_bar_arrival_action`的定义如下：

```python
def add_bar_arrival_action(self, action, period=None, instrument_ids=None,
    ohlc_kind=OhlcGeneratorConstants.time_based, name=''):
```

我们使用`period` 和`ohlc_kind`参数来指定需要等待哪种类型的K线，并使用`instrument_ids`来指定需要等待哪些交易品种的K线。
`action`参数用来指定当这些K线都增加了一根的时候，调用哪个回调函数。我们也可以指定等待不同组合的K线的回调函数都为同一个，
这时，可以使用`name`参数来区分是哪一种组合。

以下代码展示了`add_bar_arrival_action`的使用方式，其中`on_multiple_bars`就是所指定的回调函数。它的注释中说明了这个
回调函数所需要的参数及其意义。


```python
from ctxalgoctp.ctp.backtesting_utils import *


class TwoInstrumentStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(self, instrument_ids, parameters, base_folder,
                                  periods=periods, description=description, logger=logger)

        # Setup the the callback so self.on_multiple_bars will be invoked every time
        # when new ohlc bars for all traded instruments arrive.
        self.add_bar_arrival_action(self.on_multiple_bars)

    def on_multiple_bars(self, ohlc_kind, period, name, ohlcs):
        """
        :param ohlc_kind: string, the ohlc kind, can be 'time-based', 'volatility-based'.
        :param period: Periodicity in case of time-based ohlc, or int/float as volatility threshold
            in case of volatility-based ohlc.
        :param name: string, the bar arrival action name, same as the name parameter specified here.
        :param ohlcs: dict{string: OHLC}, the list of ohlcs, keys are instrument ids, values are the ohlc objects.
        """
        # Print out the ohlc bars for each traded instrument.
        # You can see that both ohlcs increase their length by one.
        print(' '.join(['{} has {} bars.'.format(sid, len(ohlcs[sid])) for sid in ohlcs]))


def main():
    start_date = '2015-01-01'
    end_date = '2015-01-31'

    config = {
        'instrument_ids': ['cu99', 'rb99'],  # Specify multiple instrument ids to trade.
        'periods': [Periodicity.FIVE_MINUTE]
    }

    backtest(TwoInstrumentStrategy, config, start_date, end_date)


if __name__ == '__main__':
    main()


```
