---
title: 在给定的时间点进入仓位
layout: post
category: live
language: zh
---

本程序展示如何在给定的时间点进入仓位。在`main`方法中的示例命令行的意思是：在14:00:00 进入cu1610的2手多仓，在14:00:01
进入SR611的2手空仓。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class TradeAtTimer(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) > 0
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.trade_count = 0
        self.trade_times = []
        self.target_positions = {}
        for sid, trade_time in self.parameters.trade_times:
            pos = self.parameters.positions[sid]['position']
            self.target_positions[trade_time] = pos
            self.trade_times.append(trade_time)
        self.trade_times = parameters.trade_times

    def on_before_run(self, strategy):
        display_positions(self)
        self.add_timer(trigger_time=self.trade_times, action=self.on_timer)
        self.add_timer(countdown=5, action=self.on_check_progress)

    def on_check_progress(self, trigger_time, supposed_trigger_time, timer_name):
        # Check trading progress.
        if self.trade_count == len(self.target_positions):
            if not self.has_pending_order():
                self.set_should_exit(True)

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        print('Timer: {}'.format(supposed_trigger_time.time()))
        instrument_id, position = self.target_positions[supposed_trigger_time.time()]
        self.change_position_to(position, instrument_id=instrument_id)
        self.trade_count += 1

    def on_order_dropped(self, kind, action, instrument_id, actual_instrument_id, reason, drop_reason):
        print('Order dropped: kind: {}, action: {}, instrument: {}, actual instrument: {}, reason: {}'.format(
            kind, action, instrument_id, actual_instrument_id, reason))


def main():
    cmd_options = '--account simnow_future3 --name test.s1 --positions cu1610:2,SR611:-2 --times cu1610:140000,SR611:140001'
    parser = get_command_line_parser(strategy_class=TradeAtTimer, cmd_options=cmd_options)

    parser.add_option('--times', type='string', dest='times', default=None,
                      help='Specify the time points to execute the trades defined by the --positions parameter. '
                           'Format is --times instrument_id:HHMMSS,instrument_id:HHMMSS. For each position defined in '
                           ' --positions, you need to specify a time for that position to get executed.')

    options = parser.parse()
    positions = options.get_positions()
    trade_times = {}
    for part in options.times:
        parts = part.split(':')
        sid = parts[0]
        trade_time = datetime.strptime(parts[1], '%H%M%S').time()
        trade_times[sid] = trade_time

    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'trade_times': trade_times,
            'positions': positions,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(TradeAtTimer, config, options)


if __name__ == '__main__':
    main()

```
