---
title: 建立CTP连接
layout: post
category: live
language: zh
---

本策略连接上CTP的行情和交易服务器，等待一段时间后，退出。

在`main`函数中，你需要通过`--account`来修改CTP交易账户的名称，同时通过`--instruments`来指定需要交易的期货品种。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class JustConnect(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_local_bookkeep(self.parameters.local_bookkeep)
        self.set_should_terminate_after_market_close(False)

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
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1703'
    parser = get_command_line_parser(strategy_class=JustConnect, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'exit_in_second': 10,
            'local_bookkeep': options.local_bookkeep,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
        }

    run_live(JustConnect, config, options)


if __name__ == '__main__':
    main()

```
