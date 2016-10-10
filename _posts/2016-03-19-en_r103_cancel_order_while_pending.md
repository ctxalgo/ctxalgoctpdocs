---
title: Cancel an order when it is pending
layout: post
category: live
language: en
---

This strategy places a difficult to execute order, and cancel it while it is pending.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class CancelOrderWhilePending(AbstractStrategy):
    def on_before_run(self, strategy):
        assert len(self.instrument_ids()) == 1
        # We can also store the order ref information in self.context.
        self.context.orders = None

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and self.context.orders is None:
            # Create a very low price related to current price to make the order hard to execute.
            price = tick['last_price'] - self.tick_size(instrument_id=instrument_id) * 30
            self.context.orders = self.change_position_to(1, price=price)
            # The tuple (10, False) means that the countdown timer is triggered in 10 seconds,
            # and it is NOT a repeated timer, so it is only triggered once.
            self.add_timer(countdown=(10, False), action=self.on_time_to_cancel_order)
        elif self.context.orders is not None:
            self.set_should_exit(not self.has_pending_order())

    def on_time_to_cancel_order(self, trigger_time, supposed_trigger_time, timer_name):
        self.cancel_orders(self.context.orders)

    def on_cancel_order(self, order_ref, instrument_id):
        print('Try to cancel: {}, {}'.format(instrument_id, order_ref))

    def on_order_canceled(self, order_ref, instrument_id):
        print('Cancelled: {}, {}'.format(instrument_id, order_ref))


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(strategy_class=CancelOrderWhilePending, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(CancelOrderWhilePending, config, options)


if __name__ == '__main__':
    main()

```
