---
title: Position change at every tick
layout: post
category: live
language: en
---

This example demonstrates how to change position of a given instrument at every tick of that instrument.
When the next tick arrives, orders from the previous `change_position_to` operation may be still pending.
Those pending orders will be canceled.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class VeryFastRepeatedChangePosition(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.traded_times = 0
        self.sign = 1

    def on_before_run(self, strategy):
        display_positions(self)

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and self.traded_times < self.parameters.trade_times:
            self.change_position_to(1 * self.sign, instrument_id=sid)
            self.sign *= -1
            self.traded_times += 1
        elif self.traded_times == self.parameters.trade_times:
            if not self.has_pending_order():
                self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future3 --name test.s1 --instruments cu1610 --trade-times 5'
    parser = get_command_line_parser(strategy_class=VeryFastRepeatedChangePosition, cmd_options=cmd_options)
    parser.add_option('--trade-times', type='int', dest='trade_times', default=5,
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

    run_live(VeryFastRepeatedChangePosition, config, options)


if __name__ == '__main__':
    main()

```
