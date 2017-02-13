---
title: Simple trend following strategy using two moving averages
layout: post
category: root
language: en
---

This example shows how to write trading strategies for a single instrument. The example includes
a double moving average trend following strategy, the code to perform backtesting of the strategy, and
the code to investigate trades through generated charts.

```python
import talib
import numpy as np
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *
```

The following listing shows the strategy class.

```python
class TrendFollowingStrategy(AbstractStrategy):
    """
    A single double moving average trend following strategy:
    1. If the fast moving average up-cross the slow moving average, change position to 1.
    2. If the fast moving average down-cross the slow moving average, change position to -1.
    That is, the position can be 0, -1 and 1.
    """
    def on_bar(self, instrument_id, bars, tick):
        # Check if there is enough ohlc bars to calculate the slow moving average because it needs
        # more bars than the fast moving average.
        # Use the ohlc(instrument_id=...) method to get the ohlc. Since we are trading a single instrument in
        # this strategy, you can omit specifying the instrument id here. But when you are trading multiple
        # instruments, you always need to specify the instrument id, otherwise, the strategy won't know
        # which instrument's ohlc to return to you. This rule also applies to other methods such as
        # in_market_period and change_position_to used below. When you invoke a method from the strategy class,
        # check its definition to see if you need to include instrument_id or not.
        if self.ohlc(instrument_id=instrument_id).length >= self.parameters.slow_ma_period:
            # Calculate the two moving averages.
            ma_fast = talib.SMA(np.array(self.ohlc().closes), timeperiod=self.parameters.fast_ma_period)
            ma_slow = talib.SMA(np.array(self.ohlc().closes), timeperiod=self.parameters.slow_ma_period)

            # Check if fast moving average up-crosses or down-crosses slow moving average.
            # cross_direction return 1 if ma_fast up-crosses ma_slow, -1 if ma_fast down-crosses ma_fast,
            # and 0 otherwise. The strategy uses this signal to change its position of the traded instrument.
            signal = cross_direction(ma_fast, ma_slow)
            # Use in_market_period to check if there should be enough time for the order to get executed on
            # current trading day. You can omit the instrument_id=... part in current strategy,
            # But when you are trading multiple instruments, you always need to specify the instrument id.
            if signal != 0 and self.in_market_period(instrument_id=instrument_id, delta=timedelta(minutes=20)):
                # The change_position_to method handles open and close positions for you.
                # It will maintain one direction positions: either long or short, not both.
                # As an example, if you start with 0 position, and then do the following sequence:
                # change_position_to(1)   # open a long position
                # change_position_to(0)   # close the long position
                # change_position_to(-1)  # open a short position
                # change_position_to(1)   # close the short position, and open a long position.
                # So at the end, you have 1 long position.
                position = 1 * signal
                # You can omit the instrument_id=... part in current strategy,
                # but when you are trading multiple instruments, you always need to specify instrument id.
                self.change_position_to(position, instrument_id=instrument_id)

```

Now, we create configurations for backtesting the strategy. After backtesting, we generate a chart in form of
HTML page to view all the traded. You can review the trades inside a browser.

```python
def main():
    start_date = '2014-12-10'  # Backtesting start date.
    end_date = '2016-12-31'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['cu00'],                      # We are trading this future instrument.
        'periods': [Periodicity.FIFTEEN_MINUTE],
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        }
    }
    start_time = datetime.now()
    report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)
    end_time = datetime.now()
    print('Backtesting duration: ' + str(end_time - start_time))

    # Use charting facility to visualize trades.
    c = ChartsWithReport(report, data_source, folder=report.base_folder, open_in_browser=True)
    c.set_instrument(config['instrument_ids'][0])
    c.period(config['periods'][0])

    c.ohlc()                                        # Draw ohlc bars.
    c.volume()                                      # Draw volumes bars.

    c.ma(config['parameters']['fast_ma_period'])    # Draw the fast moving average.
    c.ma(config['parameters']['slow_ma_period'])    # Draw the slow moving average.

    c.drawdowns(max_drawdown_color='pink')          # Visualize the max-drawdown period(s).
    c.instrument_positions()                        # Draw the current positions that we are holding.
    c.transactions()                                # Draw all the transactions (open and close of a position).
    c.balance()                                     # Draw the balance.
    c.show()                                        # Open the html page in browser.

if __name__ == '__main__':
    main()

```
