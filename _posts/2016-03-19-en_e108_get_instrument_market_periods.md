---
title: Getting market periods for instruments
layout: post
category: root
language: en
---


An instrument usually trades for multiple market periods during a trading day. For example, from 9:30 to 11:30 and
from 13:00 to 15:00. You can use the `market_periods` method to get the start and end time points for an instrument.

The following strategy first gets market periods for all the traded instruments, and then add a timer which will trigger
5 minutes before the market ends for an instrument. This is useful, for example, to flat all positions right before
market close, for intra-day strategies.


```python
from ctxalgoctp.ctp.backtesting_utils import *


class GetInstrumentMarketPeriod(AbstractStrategy):
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

    def on_before_run(self, strategy):
        # Getting the market periods for instruments. There can be multiple market periods during a trading day.
        for sid in self.instrument_ids():
            market_periods = self.market_periods(instrument_id=sid)
            print('{}: {}'.format(sid, market_periods))

            # Add a timer 5 minutes before market close for this instrument.
            # This timer callback is useful for example, to flat all positions for this instrument
            # right before market close, for a intra-day stratgey.
            self.add_timer(
                trigger_time=market_periods[-1][-1] - timedelta(minutes=5),
                on_time_action=partial(self.on_before_market_close, sid))

    def on_before_market_close(self, instrument_id, trigger_time, supposed_trigger_time, timer_name):
        print('{}: right before market close: {}'.format(instrument_id, trigger_time))


start_date = '2014-01-01'
end_date = '2014-12-31'

config = {
    'instrument_ids': ['IF99', 'cu99'],
    'strategy_period': Periodicity.FIVE_MINUTE,
    'parameters': {}
}

report, data_source = backtest(GetInstrumentMarketPeriod, config, start_date, end_date)


```
