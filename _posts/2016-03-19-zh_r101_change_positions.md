---
title: 达到给定期货品种的指定仓位
layout: post
category: live
language: zh
---

本策略会根据给定品种的指定仓位，通过交易在当前账户达到这些仓位。

在`main`函数中，你需要通过修改`--positions`来指定仓位信息。

```python
--positions cu1610:1,SR611:-2
```

表示cu1610品种需要达到多仓1，SR611品种需要达到空仓-2。



```python
from ctxalgoctp.ctp.live_strategy_utils import *


class ChangePosition(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_local_bookkeep(self.parameters.local_bookkeep)
        self.set_should_terminate_after_market_close(True)
        self.tick_count = 0
        self.last_tick_count_time = None
        self.tick_count2 = 0
        self.last_tick_count_time2 = None

    def on_before_run(self, strategy):
        display_positions(self)
        self.context.has_traded = False
        self.context.trade_started = False
        if self.parameters.trade_time is None:
            self.context.trade_started = True
            self.add_timer(countdown=1, action=self.on_timer)
        else:
            self.context.trade_started = False
            t = datetime.strptime(self.parameters.trade_time, '%H%M%S').time()
            tt = datetime.combine(self.trading_day(), t)
            self.add_timer(trigger_time=tt, action=self.on_trade_start)
            self.log('TRADE_TIMER_SET', {'trigger_time': str(tt)})

    def on_after_run(self, strategy):
        self.log('AFTER_RUN', {
            'tick_count': self.tick_count,
            'tick_count2': self.tick_count2,
            'last_tick_count_time': str(self.last_tick_count_time),
            'last_tick_count_time2': str(self.last_tick_count_time2),
        })

    def on_trade_start(self, trigger_time, supposed_trigger_time, timer_name):
        self.log('TIMER_TRIGGERED', {
            'trigger_time': str(trigger_time),
            'supposed_trigger_time': str(supposed_trigger_time),
            'timer_name': timer_name
        })
        self.context.trade_started = True
        self.add_timer(countdown=1, action=self.on_timer)

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        if not self.position_difference(self.parameters.target_positions):
            self.set_should_exit(True)

    def on_order_insert(self, order_info):
        self.log_warning(message=order_info['error_message'], data=order_info)

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if sid in self.parameters.target_positions:
            self.tick_count2 += 1
            self.last_tick_count_time2 = self.now()
        if self.context.trade_started:
            if sid in self.parameters.target_positions and not self.has_pending_order(instrument_id=sid):
                self.tick_count += 1
                self.last_tick_count_time = self.now()
                self.change_position_to(self.parameters.target_positions[sid]['position'], instrument_id=sid)
                self.context.has_traded = True


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --positions cu1703:0'  # --loggers console,file,tcp://139.196.234.169 --trade-time 210500'
    parser = get_command_line_parser(strategy_class=ChangePosition, cmd_options=cmd_options)
    parser.add_option(
        '--trade-time', type='string', dest='trade_time', default=None,
        help='Trade time in HHMMSS to execute the trades.')

    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'target_positions': options.get_positions(),
            'local_bookkeep': options.local_bookkeep,
            'trade_time': options.trade_time
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }

    run_live(ChangePosition, config, options)


if __name__ == '__main__':
    main()

```
