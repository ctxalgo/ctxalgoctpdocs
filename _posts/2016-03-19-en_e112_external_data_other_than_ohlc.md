---
title: Use external data other than ohlc and ticks
layout: post
category: root
language: en
---

This example shows how to use external data other than ohlc bars, and ticks. When the strategy needs data other than
ohlc and ticks, it needs to provide the external data through the `context` object so the strategy can use
the external data while trading.


```python
from ctxalgoctp.ctp.backtesting_utils import *


class ExternalDataStrategy(AbstractStrategy):
    def on_bar(self, instrument_id, bars, tick):
        last_bar_time = self.ohlc().dates[-1]

        # Check if we have trading signal at current ohlc bar timestamp.
        if last_bar_time in self.context['signals']:
            signal = self.context['signals'][last_bar_time]
            self.change_position_to(signal)

    def on_tick(self, instrument_id, tick):
        if self.ctp_factory.is_backtesting():
            # In backtesting mode, since all the signals data is provided via external data,
            # no need to calculate new signals.
            pass
        else:
            # In live trading mode, calculate signals from tick and store it in self.context['signals']
            # We leave this empty in this tutorial, but in live trading mode, you need to implement the calculation.
            pass


def main():
    start_date = '2016-01-01'
    end_date = '2016-01-31'

    config = {
        'instrument_ids': ['cu00'],
        'periods': [Periodicity.FIFTEEN_MINUTE],
    }

    # We specify a signals dict to contain external data needed by the strategy. Here, the trading signals are fake,
    # but you can imagine that this signals to be, say twap data, calculated from historical tick data and provided
    # through `context` for the strategy to consume.
    context = {
        'signals': {
            datetime(2016, 1, 7, 10, 45): 1,
            datetime(2016, 1, 7, 11, 15): -1
        }
    }

    # Set the `context` containing external data, here the `signals` dict before backtesting.
    backtest(ExternalDataStrategy, config, start_date, end_date, context=context)


if __name__ == '__main__':
    main()

```
