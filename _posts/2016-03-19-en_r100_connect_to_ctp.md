---
title: Connecting to CTP
layout: post
category: live
language: en
---

This strategy connects to the CTP market and trading server, wait for a few seconds and then terminates.

Inside the `main` method, you should change the values of the account `--account` and the value of the instruments
`--instruments` that you which you want to trade.


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class JustConnect(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_local_bookkeep(self.parameters.local_bookkeep)
        self.set_should_terminate_after_market_close(False)
        self.set_is_trading_enabled(self.parameters.trading_enabled)
        self.set_is_market_enabled(self.parameters.market_enabled)

    def on_before_run(self, strategy):
        # Display current holding positions in connected trading account.
        display_positions(self)
        # Add a timer to tell the strategy to terminate in exit_in_second.
        self.add_timer(countdown=self.parameters.exit_in_second, action=self.on_timer)
        # self.add_timer(countdown=5, action=self.on_timer2)

    def on_tick(self, instrument_id, tick):
        print('{} Tick: {}, {}, {}, {}, {}'.format(
            self.strategy_signature(),
            instrument_id, tick['timestamp'], self.now(), tick['last_price'], tick['volume']))

    def on_bar(self, instrument_id, bars, tick):
        for kind, data in bars.items():
            for period, data2 in data.items():
                ohlc = self.ohlc(instrument_id=instrument_id, kind=kind, period=period)
                print('{} Bar: {}, {}, {}, {}, {}'.format(
                    self.strategy_signature(),
                    kind, Periodicity.name(period), instrument_id, ohlc.dates[-1], ohlc.closes[-1]))

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(True)


def main():
    # Pass the following command line options to the strategy.
    # You need to change the account information and the instrument ids.
    cmd_options = '--account simnow_future --name test.s1 --instruments cu1705'
    parser = get_command_line_parser(strategy_class=JustConnect, cmd_options=cmd_options)
    parser.add_option(
        '--exit', type='int', dest='exit', default=10,
        help='Exit strategy in the given seconds. ')
    parser.add_option(
        '--no-trading', action='store_false', dest='trading_enabled', default=True,
        help='Should trading be enabled.')
    parser.add_option(
        '--no-market', action='store_false', dest='market_enabled', default=True,
        help='Should market be enabled.')

    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'exit_in_second': options.exit,
            'trading_enabled': options.trading_enabled,
            'market_enabled': options.market_enabled,
            'local_bookkeep': options.local_bookkeep,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
        }

    run_live(JustConnect, config, options)


if __name__ == '__main__':
    main()

```
