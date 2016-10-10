---
title: 在有还未执行完的订单的时候结束策略
layout: post
category: live
language: zh
---

本策略下了一个订单后，在该订单还没有执行完的时候，这个策略就结束了。你可以使用
[这里的代码]({% post_url 2016-03-19-zh_r106_cancel_all_pending_orders_from_previous_session %})来撤销这个还没有执行完的订单。
这两个策略一起用来展示如何撤销另一个策略里留下的订单。


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
