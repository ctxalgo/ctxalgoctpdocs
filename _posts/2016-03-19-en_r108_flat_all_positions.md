---
title: Flat all positions in a trading account.
layout: post
category: live
language: en
---

This strategy flat all positions in a trading account.

Note that for this strategy, you don't need to specify which instruments to trade through the `--instrument` command
line options. This is because before connecting to the trading account, you will not know which positions
that account holds. The strategy will automatically register for market data for instruments that are held by
the current trading account and then when their tick data arrives, you can place orders as usual.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class FlatAllPositions(AbstractStrategy):
    def on_before_run(self, strategy):
        display_positions(self)
        self.context.traded = {}
        self.add_timer(countdown=1, action=self.on_timer)
        self.context.has_traded = {}

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(not self.has_pending_order() and not self.has_position())

    def on_tick(self, instrument_id, tick):
        if self.has_position(instrument_id) and instrument_id not in self.context.has_traded:
            self.change_position_to(0, instrument_id=instrument_id)
            self.context.has_traded[instrument_id] = True


def main():
    cmd_options = '--account simnow_future4 --name test.s1'
    parser = get_command_line_parser(strategy_class=FlatAllPositions, cmd_options=cmd_options)
    options = parser.parse()
    print(options.get_base_folder())
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }
    run_live(FlatAllPositions, config, options)

if __name__ == '__main__':
    main()

```
