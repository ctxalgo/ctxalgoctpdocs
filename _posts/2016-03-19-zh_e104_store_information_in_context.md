---
title: 在context中储存跨交易日数据
layout: post
category: root
language: zh
---

CTX系统中的回测模块每天会重新创建一个交易策略的对象，这样做的目的是模拟策略实盘运行是的情景。在实盘运行时，
一般会在开盘前启动策略，收盘后结束策略。

每次创建策略对象，之前存放在对象中的数据就会丢失。但有时候，我们需要将一些数据保存才策略中，然后在之后的交易日中使用。
比如，我们想记录已经持有一些头寸的交易日的数目，到达最大数目后，我们进行平仓。

我们可以使用`self.context`来保存这些数据。保存在`self.context`中的数据会在跨交易日的时候依然可以访问。`self.context`的
使用方法和dict一致。你也可以通过`self.context.key`和`self.context.key = value`来读取和设置`self.context`的内容。

请注意，`self.context`不能在策略对象的`__init__`中使用，因为在`__init__`中`self.context`还没有被初始化。 `self.context`
可以在交易策略开始运行之后使用。

在以下的代码中，我们对之前双均线趋势跟踪策略做了如下扩展：当持仓到达一定天数的时候，我们会平掉所有头寸。我们在
`on_before_run`方法中对`self.context`进行操作。


```python
from ctxalgolib.ta.online_indicators.moving_averages import MovingAveragesIndicator
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *


class TrendFollowingWithMaxHoldingDayStrategy(AbstractStrategy):
    """
    A single double moving average trend following strategy:
    1. If the fast moving average up-cross the slow moving average, change position to 1.
    2. If the fast moving average down-cross the slow moving average, change position to -1.
    That is, the position can be 0, -1 and 1.
    """
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

        self.ma_ind1 = MovingAveragesIndicator(period=self.parameters.fast_ma_period)
        self.ma_ind2 = MovingAveragesIndicator(period=self.parameters.slow_ma_period)

        # We register the on_before_run action, this action will be invoked
        # every trading day before the trading starts.
        self.on_before_run_actions.append(self.on_before_run)

    def on_before_run(self, strategy):
        if self.has_position():
            # If we have positions, we increase the day counter.
            # Since on_before_run will be called only once a day, the counter then records the
            # number of days we have held some position.
            if 'holding_days' in self.context:
                self.context['holding_days'] += 1

    def on_bar(self, instrument_id, bars, tick):
        if self.ohlc().length >= self.parameters.slow_ma_period:
            ma_fast = self.ma_ind1.calculate(self.ohlc())
            ma_slow = self.ma_ind2.calculate(self.ohlc())
            signal = cross_direction(ma_fast, ma_slow)
            if signal == 0:
                if self.has_position():
                    # Check if we have already held the position for long enough,
                    # if so, flat the position.
                    if self.context.holding_days == self.parameters.max_holding_days:
                        self.change_position_to(0)
            else:
                self.change_position_to(signal)
                self.context['holding_days'] = 1
```

现在我们对以上交易策略进行历史数据的回测。


```python
def main():
    start_date = '2014-01-01'  # Backtesting start date.
    end_date = '2014-12-31'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['IF99'],                      # We are trading this future instrument.
        'periods': [Periodicity.FIFTEEN_MINUTE],   # The ohlc bar granularity on which trading happens.
        'parameters': {
            'fast_ma_period': 8,    # The parameter for the fast moving average.
            'slow_ma_period': 15,   # The parameter for the slow moving average.
            'max_holding_days': 2   # The maximum number of days we should hold a position.
        }
    }

    backtest(TrendFollowingWithMaxHoldingDayStrategy, config, start_date, end_date)


if __name__ == '__main__':
    main()

```
