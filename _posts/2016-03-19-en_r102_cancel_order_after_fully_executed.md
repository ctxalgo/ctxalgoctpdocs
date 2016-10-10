---
title: Cancel an order after it has fully executed
layout: post
category: live
language: en
---

This strategy places an order, waits for the order to be fully executed, and then cancel the order.
The order cancellation has no effect because the order has been fully executed. This strategy just tests
such a situation.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class CancelOrderAfterFullyExecuted(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder,
                 periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.orders = None

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and self.orders is None and not self.has_pending_order(instrument_id=sid):
            # Check whether sid is in self.instrument_ids() is necessarily. This is because if the account
            # holds other instruments, their tick data will arrive here as well, but we don't necessarily want
            # to trade those instruments. We only want to trade the instruments we specified through the command line.
            self.orders = self.change_position_to(1, instrument_id=sid)
            if len(self.orders) == 0:
                # This means that the current holding position for sid is 1.
                self.set_should_exit(True)

    def on_trade(self, trade_info):
        if not self.has_pending_order():
            self.cancel_orders(self.orders)
            self.add_timer(countdown=5, action=self.on_timer)

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(strategy_class=CancelOrderAfterFullyExecuted, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(CancelOrderAfterFullyExecuted, config, options)


if __name__ == '__main__':
    main()

```
