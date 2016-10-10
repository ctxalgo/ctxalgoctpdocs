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
    def on_before_run(self, strategy):
        display_positions(self)
        self.context.traded = {}
        self.add_timer(countdown=1, action=self.on_timer)
        self.context.has_traded = {}

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        self.set_should_exit(not self.has_pending_order() and not self.has_position())

    def on_tick(self, instrument_id, tick):
        ts = tick['timestamp']
        if self.in_market_period(ts, instrument_id=instrument_id):
            if self.has_position(instrument_id) and instrument_id not in self.context.has_traded:
                self.change_position_to(0, instrument_id=instrument_id)
                self.context.has_traded[instrument_id] = True


def main():
    cmd_options = '--account simnow_future --name test.s1'
    parser = get_command_line_parser(strategy_class=FlatAllPositions, cmd_options=cmd_options)
    options = parser.parse()
    print(options.get_base_folder())
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
    }
    run_live(FlatAllPositions, config, options)

if __name__ == '__main__':
    main()

```
