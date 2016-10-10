---
title: Exit strategy while an order is pending
layout: post
category: live
language: en
---

This strategy places an order and while the order is pending, the strategy terminates, leaving the pending order.

After this strategy terminates, you can use [here]({% post_url 2016-03-19-en_r106_cancel_all_pending_orders_from_previous_session %})
to cancel the pending order left by this strategy. These two strategies are designed as a pair to show how to cancel
orders issued by other strategies.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class PlacePendingOrderAndExit(AbstractStrategy):
    def on_before_run(self, strategy):
        self.context.traded = False

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and not self.context.traded:
            price = tick['last_price'] - self.tick_size(instrument_id=sid)*30
            self.open_limit_order(1, price=price)
            self.add_timer(countdown=5, action=self.on_time_to_exit)
            self.context.traded = True

    def on_time_to_exit(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(strategy_class=PlacePendingOrderAndExit, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(PlacePendingOrderAndExit, config, options)


if __name__ == '__main__':
    main()

```
