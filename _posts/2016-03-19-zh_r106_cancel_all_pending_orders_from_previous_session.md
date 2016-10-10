---
title: 撤销在同一交易账户的其它策略留下来的订单。
layout: post
category: live
language: zh
---

本策略撤销在同一交易账户上其它策略留下来的订单。要做到这一点，本策略首先进行订单查询。该查询返回在同一交易账户上的
当前交易日上发生的所有订单。在此基础上本策略收集所有还未完成的订单，然后撤销他们。然后通过`on_external_order`来
获得被撤销了的订单的状态。这是因为这些订单不是本策略发出的，所以`on_order`，`on_trade`等回调函数是不会被调用的。

要查看本策略的运行效果，你需要先使用[这里的代码]({% post_url 2016-03-19-zh_r105_place_pending_order_and_exit %})来下一个订单。
在那个策略结束后，马上运行本策略来撤销在这个订单。


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
