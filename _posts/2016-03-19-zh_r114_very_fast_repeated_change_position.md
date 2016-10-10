---
title: 在每一个tick到来时改变仓位
layout: post
category: live
language: zh
---

本程序展示如何在每一tick到来时改变仓位。当一个tick到来时，上一次`change_position_to`发生的报单不一定已经执行完毕了。
那些没有执行的报单会被撤单。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class VeryFastRepeatedChangePosition(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.traded_times = 0
        self.sign = 1

    def on_before_run(self, strategy):
        display_positions(self)

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.instrument_ids() and self.traded_times < self.parameters.trade_times:
            self.change_position_to(1 * self.sign, instrument_id=sid)
            self.sign *= -1
            self.traded_times += 1
        elif self.traded_times == self.parameters.trade_times:
            if not self.has_pending_order():
                self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future3 --name test.s1 --instruments cu1610 --trade-times 5'
    parser = get_command_line_parser(strategy_class=VeryFastRepeatedChangePosition, cmd_options=cmd_options)
    parser.add_option('--trade-times', type='int', dest='trade_times', default=5,
                      help='Specify the number of trades to perform.')
    options = parser.parse()

    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'trade_times': options.trade_times,
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(VeryFastRepeatedChangePosition, config, options)


if __name__ == '__main__':
    main()

```
