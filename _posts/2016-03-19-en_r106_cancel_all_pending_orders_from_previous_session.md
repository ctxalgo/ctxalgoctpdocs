---
title: Cancel orders issued by other strategies on the same trading account
layout: post
category: live
language: en
---

This strategy cancels all pending orders that are issued by other strategies connecting to the same trading account.
It does so by first querying orders that are issued on the trading account on current trading day, and finding out
which orders are pending, and cancelling those pending orders. To check if the orders are canceled, the strategy
relies on the `on_external_order` callback, since those orders are not issued by current strategies, the normal
`on_order`, `on_trade` callbacks are not invoked.

To see how it works, you can use [here]({% post_url 2016-03-19-en_r105_place_pending_order_and_exit %}) to leave a pending
order, and then run this strategy immediately after to cancel that pending order.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class CancelAllPendingOrders(AbstractStrategy):
    def on_before_run(self, strategy):
        self.context.pendings = []
        self.query_order()

    def on_query_order(self, data):
        if not(data['order_status'] in [OrderStatus.AllTraded, OrderStatus.Canceled]):
            # The query order callback will give you all orders that happens in the trading account
            # from the beginning of current trading day. Here we only collect orders that are still pending.
            self.context.pendings.append(data)

        if data['is_last']:
            # `is_last` tells us that there will be no more `on_query_order` callbacks anymore,
            # so we have collected all pending orders. Then we cancel those orders.
            self.cancel_external_orders(self.context.pendings)
            self.add_timer(countdown=1, action=self.on_check_progress)

    def on_check_progress(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(len(self.context.pendings) == 0)

    def on_external_order(self, order_info):
        if order_info['status'] in [OrderStatus.AllTraded, OrderStatus.Canceled]:
            # For an external order status change, we check if its status is finished,
            # if so, we remove it from our list of pending orders.
            self.context.pendings = [o for o in self.context.pendings if not self.same_external_order(o, order_info)]


def main():
    # cmd_options = '--account simnow_future4 --name test.s1'
    cmd_options = None
    parser = get_command_line_parser(strategy_class=CancelAllPendingOrders, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(CancelAllPendingOrders, config, options)

if __name__ == '__main__':
    main()

```
