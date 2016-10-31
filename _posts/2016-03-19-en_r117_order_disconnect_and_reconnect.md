---
title: Trading server disconnection and re-connection during an order execution.
layout: post
category: live
language: en
---

This strategy tests the following scenario:
1. place an order.
2. disconnect from trading server immediately, so the strategy cannot receive RTN_ORDER, RTN_TRADE callbacks.
3. re-connect to the trading server after a few seconds.
4. receive RTN_ORDER, RTN_TRADE callbacks from the previous trading connection.

This scenario simulates the case when the connection to CTP trading server gets disconnected and then the CTP
reconnects. We need to make sure that we can continue to get trading related callbacks after reconnection.


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
