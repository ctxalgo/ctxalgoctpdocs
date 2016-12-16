---
title: Reaching given positions
layout: post
category: live
language: en
---

This strategy tries to reach the given positions for certain instruments.

Inside the `main` method, you should change the values of the account `--account` and change `--positions` to specify
the instruments and their positions to reach.

```python
--positions cu1610:1,SR611:-2
```

means to reach one long position for instrument cu1610, and -2 position for instrument SR611.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class ChangePosition(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_local_bookkeep(self.parameters.local_bookkeep)

    def on_before_run(self, strategy):
        display_positions(self)
        self.add_timer(countdown=1, action=self.on_timer)
        self.context.has_traded = False
        self.context.trade_started = False
        if self.parameters.trade_time is None:
            self.context.trade_started = True
        else:
            self.context.trade_started = False
            t = datetime.strptime(self.parameters.trade_time, '%H%M%S').time()
            tt = datetime.combine(self.trading_day(), t)
            self.add_timer(trigger_time=tt, action=self.on_trade_start)

    def on_trade_start(self, trigger_time, supposed_trigger_time, timer_name):
        self.context.trade_started = True

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        if not self.position_difference(self.parameters.target_positions):
            self.set_should_exit(True)

    def on_order_insert(self, order_info):
        self.log_warning(message=order_info['error_message'], data=order_info)

    def on_tick(self, instrument_id, tick):
        if self.context.trade_started:
            sid = instrument_id
            if sid in self.parameters.target_positions and not self.has_pending_order(instrument_id=sid):
                self.change_position_to(self.parameters.target_positions[sid]['position'], instrument_id=sid)
                self.context.has_traded = True


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --positions a1701:1'
    parser = get_command_line_parser(strategy_class=ChangePosition, cmd_options=cmd_options)
    parser.add_option(
        '--trade-time', type='string', dest='trade_time', default=None,
        help='Trade time in HHMMSS to execute the trades.')

    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'target_positions': options.get_positions(),
            'local_bookkeep': options.local_bookkeep,
            'trade_time': options.trade_time
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(ChangePosition, config, options)


if __name__ == '__main__':
    main()

```
