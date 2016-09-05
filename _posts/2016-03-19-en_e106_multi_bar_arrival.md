---
title: Multiple-bar arrival event
layout: post
category: root
language: en
---

In a strategy that trades multiple instrument, sometimes you want to have a callback when all the traded instruments
have advanced by one bar. For example, in a pair trading strategy, you want to access some arbitrage possibility when
both the instruments have a new bar arrived. In this case, you can use the `add_bar_arrival_action` method in the
strategy class.

The `add_bar_arrival_action` method has the following signature:

```python
def add_bar_arrival_action(self, action, period=None, instrument_ids=None,
    ohlc_kind=OhlcGeneratorConstants.time_based, name=''):
```

You use the `period` and `ohlc_kind` parameter to specify what kinds of ohlc bars you are waiting for, and use the
`instrument_ids` parameter to specify which instruments' ohlc bars you are waiting for. The `action` parameter
specifies a function pointer, which will be invoked when all the required bars arrive. You can use the same
function pointer for more than one cases of multiple bar arrivals, in this case, you can use the `name` parameter
to give each case a name.

The following strategy shows an example of `add_bar_arrival_action`, and the `on_multiple_bars` method is the
callback function. Its comment explains the parameters needed for this callback.


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
