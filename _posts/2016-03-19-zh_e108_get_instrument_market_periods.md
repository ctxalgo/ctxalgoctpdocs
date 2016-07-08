---
title: 获取某个品种的交易时段
layout: post
category: root
language: zh
---


一个品种在一个交易日中通常会交易几个时段，比如从9:30到11:30，然后从13:00到15:00 。你可以使用`market_periods`方法来
获取某个品种在某个交易日内的交易时段。

以下示例首先获取交易品种的交易时段，然后在每个交易品种收盘的5分钟前设置一个定时器。在日内的交易策略中，这样的定时器可以用来
对所有持仓进行平仓操作。


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
