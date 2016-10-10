---
title: A high frequency trading example
layout: post
category: live
language: en
---

This example implements a strategy which trades the given instruments into long and short positions, hold
those positions for a few seconds, and switch to the other direction. For example, it trades cu1610 in 2 long positions,
holds that position for 5 seconds, and switch to -2 short positions.

This example also shows how to add customized command line options.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class HighFrequencyBothDirectionTrade(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_should_use_remote_account_and_position(not self.parameters.local_bookkeep)
        self.order_count = 0
        self.last_trade_times = {}
        self.signs = {}
        self.hold_times = {}

    def on_before_run(self, strategy):
        self.last_trade_times = {sid: datetime.now() for sid in self.instrument_ids()}
        self.signs = {sid: 1 for sid in self.instrument_ids()}
        self.hold_times = {sid: timedelta(seconds=self.parameters.hold_periods[sid]) for sid in self.instrument_ids()}

    def on_tick(self, instrument_id, tick):
        if not self.has_pending_order() and self.order_count > self.parameters.order_count:
            self.set_should_exit(True)
        elif not self.should_exit():
            if instrument_id in self.instrument_ids() and not self.has_pending_order(instrument_id):
                hold_time = self.now() - self.last_trade_times[instrument_id]
                if hold_time > self.hold_times[instrument_id]:
                    position = self.parameters.positions[instrument_id]['position'] * self.signs[instrument_id]
                    self.change_position_to(position, instrument_id=instrument_id)
                    self.signs[instrument_id] *= -1
                    self.order_count += 1

    def on_trade(self, trade_info):
        self.last_trade_times[trade_info['instrument_id']] = self.now()


def main():
    cmd_options = '--account simnow_future3 --name test.s1 --positions cu1610:2,SR611:-2'
    parser = get_command_line_parser(strategy_class=HighFrequencyBothDirectionTrade, cmd_options=cmd_options)

    # Add new options into the strategy command line parser.
    # The way to add new options is the same as OptionParser.
    parser.add_option('--hold-periods', type='string', dest='hold_periods',
                      help=("The positions in form of instrument_id:period[,instrument_id:period]*. For example, "
                            "cu1609:5,SR609:10 means hold position of cu1609 for at least 5 seconds, and "
                            "position of SR609 for at least 10 seconds. If None, all positions are assumed to hold "
                            "for 5 seconds."))
    parser.add_option('--orders', type='int', dest='order_count', default=20,
                      help='The number of order requests.')

    options = parser.parse()
    positions = options.get_positions()
    if options.hold_periods is None:
        hold_periods = {sid: 5 for sid in positions}
    else:
        hold_periods = {x.split(':')[0]: int(x.split(':')[1]) for x in options.hold_periods.strip().split(',')}

    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'positions': positions,
            'hold_periods': hold_periods,
            'local_bookkeep': options.local_bookkeep,
            'order_count': options.order_count,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
        }

    run_live(HighFrequencyBothDirectionTrade, config, options)


if __name__ == '__main__':
    main()



```
