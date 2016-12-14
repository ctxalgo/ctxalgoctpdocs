---
title: 平掉交易账户里的所有仓位
layout: post
category: live
language: zh
---

本策略平掉连接的交易账户中的所有仓位。

注意运行这个策略的时候，你是不需要通过`--instruments`来指定所需要交易的期货品种的。这是因为在连接到交易账户之前，你也
不知道该账户上有哪些品种的仓位。策略会自动为持有仓位的所有品种订阅交易数据。等这些数据来到的时候，你就可以和往常一样
下单平仓了。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class FlatAllPositions(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, description=None, logger=None):
        AbstractStrategy.__init__(self, instrument_ids, parameters, base_folder, description=description, logger=logger)
        self.set_should_terminate_after_market_close(True)
        self.has_traded = set([])

    def on_before_run(self, strategy):
        display_positions(self)
        self.add_timer(countdown=5, action=self.on_timer)

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(not self.has_pending_order() and self.relevant_instruments() == self.has_traded)

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        ts = tick['timestamp']
        if self.in_market_period(ts, instrument_id=sid) and tick['ask_price1'] > 0 and tick['bid_price1'] > 0:
            if self.has_position(sid) and sid not in self.has_traded:
                if self.parameters.market:
                    self.change_position_to(
                        0, instrument_id=sid, market=True, cancel_in_time=self.parameters.order_timeout)
                else:
                    self.change_position_to(
                        0, instrument_id=sid, tick_delta=self.parameters.tick_delta,
                        cancel_in_time=self.parameters.order_timeout)
                self.has_traded.add(sid)


def main():
    cmd_options = '--account simnow_future4 --name test.flat_all_positions --order-timeout 10'
    parser = get_command_line_parser(strategy_class=FlatAllPositions, cmd_options=cmd_options)
    parser.add_option(
        '--order-kind', type=str, dest='order_kind', default='limit',
        help="Order kind, valid value can be 'limit' or 'market'. Default is 'limit'.")

    parser.add_option(
        '--tick-delta', type=int, dest='tick_delta', default=None,
        help="Tick delta used for limit orders. Have effect only when --order-kind is limit. "
             "If None, 1 is used.")

    parser.add_option(
        '--order-timeout', type=int, dest='order_timeout', default=None,
        help="The number of seconds for an order to timeout, when timeout, pending orders will be cancelled. "
             "None means no timeout. Default is None.")

    options = parser.parse()
    print(options.get_base_folder())
    config = {
        'base_folder': options.get_base_folder(),
        'parameters': {
            'order_timeout': options.order_timeout,
            'market': options.order_kind == 'market',
            'tick_delta': options.tick_delta
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }
    run_live(FlatAllPositions, config, options)

if __name__ == '__main__':
    main()

```
