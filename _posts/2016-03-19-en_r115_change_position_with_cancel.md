---
title: Change positions in fast pace with automatic cancellation
layout: post
category: live
language: en
---

This example changes positions on every tick. And some times the previous position changing has not finished, and
the pending orders will be canceled by the next `change_position_to`.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class ChangePositionWithCancel(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

        self.traded_times = 0
        self.sign = -1

    def on_before_run(self, strategy):
        display_positions(self)

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and self.traded_times < self.parameters.trade_times:
            if self.traded_times < self.parameters.traded_times-1:
                tick_delta = -20
            else:
                tick_delta = 1
            # When a `change_position_to` is called, if there are pending orders from the previous `change_position_to`,
            # those pending orders will be cancelled. And here, we also specify `cancel_in_time`, meaning that
            # in 5 seconds, if the orders placed by current `change_position_to` has not been fully traded,
            # those pending order will be cancelled as well.
            self.change_position_to(2 * self.sign, instrument_id=sid, cancel_in_time=5, tick_delta=tick_delta)
            self.sign *= -1
            self.traded_times += 1
        elif self.traded_times == self.parameters.trade_times and not self.has_pending_order():
                self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future3 --name test.s1 --instruments cu1610 --trade-times 3'
    parser = get_command_line_parser(strategy_class=ChangePositionWithCancel, cmd_options=cmd_options)
    parser.add_option('--trade-times', type='int', dest='trade_times', default=3,
                      help='Specify the number of trades to perform.')
    options = parser.parse()

    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'trade_times': options.trade_times,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(ChangePositionWithCancel, config, options)

if __name__ == '__main__':
    main()

```
