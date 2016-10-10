---
title: Specify cancellation when placing orders
layout: post
category: live
language: en
---

This strategy shows how to specify a time-based order cancellation when placing an order. The code uses
`open_limit_order` to place a limit order. And it uses the `cancel_in_time` parameter to specify that if
in 10 seconds the order is not fully executed, cancel the left over part. You can specify cancellation in all
order related methods, such as `open_limit_order`, `close_limit_order`, `open_market_order`, `close_market_order`,
`change_position_to` and `change_positions_to`.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class AutomaticOrderCancellation(AbstractStrategy):
    def on_before_run(self, strategy):
        assert len(self.instrument_ids()) == 1
        self.context.traded = False

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and not self.context.traded:
            if self.in_market_period(instrument_id=sid, delta=60):
                # `in_market_period` makes sure we only place an order when we are not too close to market period end.
                # In this case, there is at least 60 seconds before market period ends.
                price = tick['last_price'] - self.tick_size(instrument_id=instrument_id) * 50
                self.open_limit_order(1, price=price, cancel_in_time=10)
                self.context.traded = True
        elif self.context.traded and not self.has_pending_order():
            self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(strategy_class=AutomaticOrderCancellation, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(AutomaticOrderCancellation, config, options)


if __name__ == '__main__':
    main()

```
