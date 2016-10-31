---
title: 在订单执行过程中中断并重连CTP交易连接
layout: post
category: live
language: zh
---

当前程序测试以下场景：
1. 下单。
2. 立即断开CTP交易连接，所以RTN_ORDER，RTN_TRADE等回报无法收到。
3. 过一段时间后重新连接CTP交易连接。
4. 收取之前连接的交易回报。

这个场景用来模拟在订单执行过程中交易连接断开又由CTP自动重连的情况。


```python
from ctxalgoctp.ctp.live_strategy_utils import *


class PlaceOrderDisconnectAndReconnect(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        assert len(instrument_ids) == 1
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

        self.set_local_bookkeep(False)
        self.has_traded = False
        self.reconnected = False
        self.disconnect_time = None

    def on_tick(self, instrument_id, tick):
        sid = instrument_id
        if self.is_trading_connected():
            if sid in self.instrument_ids() and not self.has_traded:
                    # Decide whether to open or close positions for the traded instrument.
                    current_position = self.single_sided_position(instrument_id=instrument_id)
                    tick_delta = None  # Specify a negative value will make the trade more difficult to happen.
                    position = 0 if current_position is None or current_position != 0 else 2
                    self.change_position_to(position, instrument_id=instrument_id, tick_delta=tick_delta)
                    self.has_traded = True

                    # Disconnect from the trading server.
                    self.disconnect_trading()
                    self.disconnect_time = self.now()
        else:
            if not self.reconnected:
                # After trading server disconnection, wait for a few seconds and reconnect.
                if self.disconnect_time is not None and (self.now() - self.disconnect_time) > timedelta(seconds=10):
                    self.connect_trading()
                    self.reconnected = True

        # Terminate when all trades finish.
        if self.has_traded and not self.has_pending_order():
            self.set_should_exit(True)


def main():
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1610'
    parser = get_command_line_parser(PlaceOrderDisconnectAndReconnect, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'logger': options.get_logger()
    }
    run_live(PlaceOrderDisconnectAndReconnect, config, options=cmd_options)


if __name__ == '__main__':
    main()

```
