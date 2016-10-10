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
    def on_before_run(self, strategy):
        # Display current holding positions in connected trading account.
        display_positions(self)
        # Add a timer to tell the strategy to terminate in exit_in_second.
        self.add_timer(countdown=self.parameters.exit_in_second, action=self.on_timer)

    def on_tick(self, instrument_id, tick):
        print('Tick: {}, {}, {}, {}, {}'.format(
            instrument_id, tick['timestamp'], self.now(), tick['last_price'], tick['volume']))

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(True)


def main():
    # Pass the following command line options to the strategy.
    # You need to change the account information and the instrument ids.
    cmd_options = '--account simnow_future3 --name test.s1 --instruments cu1610,SR611'
    parser = get_command_line_parser(strategy_class=JustConnect, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'exit_in_second': 10
        },
        'description': options.get_description(),
        'logger': options.get_logger()
        }

    run_live(JustConnect, config, options)


if __name__ == '__main__':
    main()

```
