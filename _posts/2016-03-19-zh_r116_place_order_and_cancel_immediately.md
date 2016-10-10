---
title: 下单后立刻撤单
layout: post
category: live
language: zh
---

本程序展示如何在下单后立刻撤单。这时候该报单很可能没有被执行完。撤单的操作会取消掉还没有执行的单子。这个例子展示
如何进行撤单。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class PlaceOrderAndCancelItImmediately(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.has_traded = False

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and not self.has_traded:
            price = tick['last_price']
            # You can lower the price to make the order harder to get filled.
            # price -= self.tick_size(sid) * 10
            orders = self.open_limit_order(10, price=price)
            self.cancel_orders(orders)
            self.has_traded = True
        elif self.has_traded:
            if not self.has_pending_order():
                self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(PlaceOrderAndCancelItImmediately, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'logger': options.get_logger()
    }
    run_live(PlaceOrderAndCancelItImmediately, config, options)

if __name__ == '__main__':
    main()

```
