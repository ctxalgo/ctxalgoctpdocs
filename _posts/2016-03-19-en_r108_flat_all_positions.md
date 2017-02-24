---
title: Flat all positions in a trading account.
layout: post
category: live
language: en
---

This strategy flat all positions in a trading account.

Note that for this strategy, you don't need to specify which instruments to trade through the `--instrument` command
line options. This is because before connecting to the trading account, you will not know which positions
that account holds. The strategy will automatically register for market data for instruments that are held by
the current trading account and then when their tick data arrives, you can place orders as usual.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class FlatAllPositions(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, description=None, logger=None):
        AbstractStrategy.__init__(self, instrument_ids, parameters, base_folder, description=description, logger=logger)
        self.set_should_terminate_after_market_close(True)
        self.has_traded = set([])
        self.exit_time_reached = None

    def on_before_run(self, strategy):
        display_positions(self)
        self.add_timer(countdown=5, action=self.on_timer)
        if self.parameters['exit'] is not None:
            self.exit_time_reached = False
            self.add_timer(trigger_time=datetime.now() + timedelta(seconds=self.parameters['exit']),
                           action=self.on_exit_time)

    def on_exit_time(self, trigger_time, supposed_trigger_time, timer_name):
        self.exit_time_reached = True
        if self.has_pending_order():
            self.cancel_orders()

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        if (self.exit_time_reached is not None) and self.exit_time_reached and (not self.has_pending_order()):
            self.set_should_exit(True)
        else:
            self.set_should_exit(not self.has_pending_order() and self.relevant_instruments() == self.has_traded)

    def on_tick(self, instrument_id, tick):
        if self.exit_time_reached is None or not self.exit_time_reached:
            sid = instrument_id
            ts = tick['timestamp']
            if self.in_market_period(ts, instrument_id=sid) and tick['ask_price1'] > 0 and tick['bid_price1'] > 0:
                if self.has_position(sid) and sid not in self.has_traded:
                    self.change_position_to(
                        0, instrument_id=sid, market=self.parameters.market,
                        tick_delta=self.parameters.tick_delta, cancel_in_time=self.parameters.order_timeout)
                    self.has_traded.add(sid)


def main():
    cmd_options = '--account simnow_future --name test.flat_all_positions --order-timeout 10 --order-kind market --exit 15'
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

    parser.add_option(
        '--exit', type=int, dest='exit', default=None,
        help="The max number of seconds for the script to run. If not None, terminate the script "
             "after that amount of time, cancel pending orders if necessary.")


    options = parser.parse()
    print(options.get_base_folder())
    config = {
        'base_folder': options.get_base_folder(),
        'parameters': {
            'order_timeout': options.order_timeout,
            'market': options.order_kind == 'market',
            'tick_delta': options.tick_delta,
            'exit': options.exit
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }
    run_live(FlatAllPositions, config, options)

if __name__ == '__main__':
    main()

```
