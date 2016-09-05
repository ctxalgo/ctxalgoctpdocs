---
title: Strategy mission control
layout: post
category: root
language: en
---

This example shows how to setup mission control inside a strategy. Mission control provides a way to communicate
with a running strategy from outside. You can send messages and receives messages from a strategy in a async fashion.

To enable mission control, you need to give a `MissionControl` object to the strategy, and inside the strategy, you
need to inherit the `on_message_from_mission_controller` method. This callback method will be invoked every time
there is a message arrives from outside a strategy. And inside the strategy, you can call `send_message_to_mission_controller`
to send a message to mission controller.


```python
import threading
import zmq
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.mission_control.mission_control import MissionControl, MissionController

```

We create a strategy which redefines `on_message_from_mission_controller` with a simple logic: echo the received
messages from mission controller back to the mission controller.

```python
class StrategyWithMissionControl(TrendFollowingStrategy):
    def on_message_from_mission_controller(self, message):
        self.send_message_to_mission_controller('Echo: ' + message)

```

Here we create a thread to run the backtester, so in the main thread, we can accept user's input. Note that when we
start the backtester, we passed the `mission_control` object to it.

```python
class StrategyBacktestingThread(threading.Thread):
    """
    This thread runs backtesting on TrendFollowingStrategy.
    """
    def __init__(self, mission_control):
        threading.Thread.__init__(self)
        self.mission_control = mission_control

    def run(self):
        start_date = '2014-01-01'  # Backtesting start date.
        end_date = '2014-12-31'  # Backtesting end date.

        base_folder = safe_get_base_folder(StrategyWithMissionControl)
        config = {
            'instrument_ids': ['IF99'],
            'periods': [Periodicity.ONE_MINUTE],
            'parameters': {
                'fast_ma_period': 8,
                'slow_ma_period': 15,
            },
            'base_folder': base_folder,
            'logger': StrategyUtils.get_logger(base_folder, has_console=False)
        }
        backtest(StrategyWithMissionControl, config, start_date, end_date, mission_control=self.mission_control,
                 initial_capital=10000*10000.0)

```

Here, we start the strategy backtesting in a separate thread, and in the main thread, we read messages from console,
and send the read messages into the strategy, and display the messages from the strategy.

```python
def main():
    strategy_name = 'test.s1'
    context = zmq.Context()
    queue_from_strategy_to_mission_controller = 'tcp://127.0.0.1:6560'
    queue_from_mission_controller_to_strategy = 'tcp://127.0.0.1:6561'

    # Create a mission control object, which will be given to the strategy.
    mission_control = MissionControl(
        context,
        queue_to_mission_controller=queue_from_strategy_to_mission_controller,
        queue_from_mission_controller=queue_from_mission_controller_to_strategy,
        strategy_name=strategy_name)

    # Create a mission controller, which is outside of a strategy and sends and receives messages from a strategy.
    mission_controller = MissionController(
        context,
        queues_from_strategies=queue_from_strategy_to_mission_controller,
        queues_into_strategies=queue_from_mission_controller_to_strategy,
        strategy_name=strategy_name)

    backtester = StrategyBacktestingThread(mission_control)
    backtester.start()

    # Here you can type in some messages into console, and then receives messages echoed from the strategy.
    while backtester.is_alive():
        message = raw_input("Enter next message: ")
        # Send messages to the strategy.
        print('Send message to strategy:  ' + message)
        mission_controller.send_to_strategy(strategy_name, message)

        received = mission_controller.poll_from_strategy()
        for received_topic, received_message in received.items():
            print('Messages from strategy: {}: {}'.format(received_topic, received_message))

    backtester.join()
    mission_control.dispose()
    mission_controller.dispose()
    context.term()


if __name__ == '__main__':
    main()

```
