---
title: Place an order and cancel it immediately after
layout: post
category: live
language: en
---

This strategy places and order and immediately afterwards, cancel that order. Most likely the order will not be
fully executed, and the cancellation will cancel the remaining trades. This strategy demonstrates
how order cancellation works.


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
