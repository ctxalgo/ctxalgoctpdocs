---
title: Persist information in context across trading days
layout: post
category: root
language: en
---

The backtesting framework in CTX will create a new trading strategy object on every trading day to simulate
the fact that usually we start our trading strategy before a trading day starts, and turn it down
when the trading ends.

When a new trading strategy object is created, all the information stored in it will be gone. Sometime, you want
to keep some information across trading days. For example, in a strategy which keeps positions for multiple days,
you want to keep counting the number of days that we have already held the position and close a position when
a maximum holding length is reached.

You can use `self.context` to keep information across trading days. Use `self.context` as a dict. You can also
use the notation self.context.key and self.context.key = value to read and write to `self.context`.

Note that `self.context` is not accessible from the `__init__` method of the strategy, because `self.context`
has not been initialized in the `__init__` method.  It is accessible when the strategy starts to run.

In the following code, which is an adaption of the double moving average trend following strategy,
we mention `self.context` inside the `on_before_run` action. The idea is that if we hold a position longer than
the maximum allowed days, we flat the position.


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
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

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

Now, we create configurations for backtesting the strategy.

```python
def main():
    start_date = '2014-01-01'  # Backtesting start date.
    end_date = '2014-12-31'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['IF99'],                      # We are trading this future instrument.
        'strategy_period': Periodicity.FIFTEEN_MINUTE,   # The ohlc bar granularity on which trading happens.
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
