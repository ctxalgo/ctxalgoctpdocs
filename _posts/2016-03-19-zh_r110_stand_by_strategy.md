---
title: Stand by strategy
layout: post
category: live
language: zh
---

```python
import zmq
from ctxalgolib.data_feed.chore_server_config import ChoreServerConfig
from ctxalgoctp.ctp.live_strategy_utils import *


class StandByStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_should_use_remote_account_and_position(not self.parameters.local_bookkeep)
        self.order_count = 0
        self.last_trade_times = {}
        self.signs = {}
        self.hold_times = {}
        self.set_order_ref_starting_value(parameters['start_order_ref'])
        self.exit_requested = False
        self.set_should_terminate_after_market_close(False)

    def on_message_from_mission_controller(self, message):
        self.send_message_to_mission_controller('Received message: ' + message)
        if message == 'connect_trading':
            self.connect_trading()
        elif message == 'disconnect_trading':
            self.disconnect_trading()
        elif message == 'connect_market':
            self.connect_market()
        elif message == 'disconnect_market':
            self.disconnect_market()
        elif message == 'get_trading_account':
            self.get_trading_account()
        elif message == 'get_investor_position':
            self.get_investor_position('')
            display_positions(self)
        elif message.startswith('change_position_to'):
            parts = message.split(':')
            positions = StrategyCommandLineUtils.parse_positions(':'.join(parts[1:]))
            for sid, position_info in positions.items():
                if not self.has_pending_order(instrument_id=sid):
                    self.change_position_to(**position_info)

        elif message.startswith('get_instrument'):
            parts = message.split(':')
            sid = parts[1]
            self.get_instrument(instrument_id=sid)
        elif message.startswith('get_commission_rate'):
            parts = message.split(':')
            sid = parts[1]
            self.get_commission_rate(instrument_id=sid)
        elif message.startswith('get_margin_rate'):
            parts = message.split(':')
            sid = parts[1]
            self.get_margin_rate(instrument_id=sid)
        elif message.startswith('heartbeat'):
            self.perform_heartbeat(Topics.strategy_heartbeat_status_running(), immediately=True,
                                   include_position=True, include_account=True)
        elif message.startswith('warning'):
            parts = message.split(':')
            if len(parts) == 2:
                self.log_warning(message=parts[1])
        elif message.startswith('exception'):
            parts = message.split(':')
            if len(parts) == 2:
                raise Exception(parts[1])
        elif message == 'exit':
            self.exit_requested = True

    def on_before_run(self, strategy):
        self.add_timer(countdown=timedelta(seconds=1), action=self.on_timer)
        display_positions(strategy)

    def on_timer(self, trigger_time, supposed_trigger_time, timer_name):
        if self.exit_requested and not self.has_pending_order():
            self.set_should_exit(True)

    def on_tick(self, instrument_id, tick):
        pass
        # print('Tick data: sid={}, price={}, time={}'.format(instrument_id, tick['last_price'], tick['timestamp']))


def main():
    # TODO: To view logs from the website, set the messge_proxy_address to ChoreServerConfig().tcp_address().
    message_proxy_address = 'tcp://127.0.0.1'

    # TODO: To use an external trade execution server, set external_trade_executor to True.
    # TODO: Of course, you have to start that trade execution server, which is at
    # TODO: ctxalgoctp.ctp.trade_executors.zmq_trade_execution_server.
    external_trade_executor = True

    # TODO: If you trade locally, you have to specify the trading account.
    account = '--account simnow_future4'

    # TODO: Specify the instruments to trade.
    instruments = '--instruments cu1610,SR611'

    # local_bookkeep = '--local-bookkeep'
    local_bookkeep = ''

    if external_trade_executor:
        # The trade executor server is assumed to be at the same machine as current strategy.
        trader_address = 'tcp://127.0.0.1'
        trade_executor = '--trade-executor remote-proxy --trade-executor-address ' + trader_address
        # account = ''
    else:
        trade_executor = ''

    logger = '--loggers console,file,remote-proxy --logger-address ' + message_proxy_address
    mission_control = '--mission-control remote-proxy --mc-address ' + message_proxy_address

    cmd_options = '--name test.s1 --base-folder c:\\tmp\\s1 {} {} {} {} {} {}'.format(
        account, instruments, logger, mission_control, trade_executor, local_bookkeep)
    cmd_options = cmd_options.strip()
    context = zmq.Context()
    parser = get_command_line_parser(strategy_class=StandByStrategy, cmd_options=cmd_options, context=context)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {
            'start_order_ref': options.start_order_ref,
            'local_bookkeep': options.local_bookkeep
        },
        'description': options.get_description(),
        'logger': options.get_logger()
    }
    run_live(StandByStrategy, config, options)
    context.term()


if __name__ == '__main__':
    main()



```
