---
title: 策略任务控制
layout: post
category: root
language: zh
---

本例子展示了如何对策略进行任务控制。任务控制是一种和正在运行的策略进行通讯的手段。你可以以异步的方式，
向策略发送消息，也可以收到来自策略的消息。

要配置任务控制，你需要把一个`MissionControl`对象传递给策略，在策略内部，你需要继承`on_message_from_mission_controller`
方法。每当有消息从策略外部传送给策略的时候，这个方法就会被调用。在策略里，你可以使用`send_message_to_mission_controller`
来向策略外部的任务控制发送消息。



```python
import threading
import zmq
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.mission_control.mission_control import MissionControl, MissionController

```

在此，我们重定义`on_message_from_mission_controller`方法，然后把收到的来自外部的消息返回重新在发回给任务控制器。


```python
class StrategyWithMissionControl(TrendFollowingStrategy):
    def on_message_from_mission_controller(self, message):
        self.send_message_to_mission_controller('Echo: ' + message)

```

这里，我们建立一个线程来运行策略回测，这样，我们在主线程里就可以让用户在命令行输入信息。


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

        base_folder = os.path.join(os.environ['CTXALGO_TEST'], 'strategies', StrategyWithMissionControl.__name__)
        config = {
            'instrument_ids': ['IF99'],
            'strategy_period': Periodicity.ONE_MINUTE,
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

我们在一个单独线程里启动策略回测，然后用户可以往命令行输入消息，并且观测到来自策略的消息回放。


```python
def main():
    topic= 'eqb'
    context = zmq.Context()
    queue_from_strategy = 'tcp://127.0.0.1:6560'
    queue_into_strategy = 'tcp://127.0.0.1:6561'

    # Create a mission control object, which will be given to the strategy.
    mission_control = MissionControl(
        context, queue_from_strategy=queue_from_strategy, queue_into_strategy=queue_into_strategy, topic=topic)

    # Create a mission controller, which is outside of a strategy and sends and receives messages from a strategy.
    mission_controller = MissionController(
        context, queues_from_strategy=queue_from_strategy, queues_into_strategy=queue_into_strategy, topic=topic)

    backtester = StrategyBacktestingThread(mission_control)
    backtester.start()

    # Here you can type in some messages into console, and then receives messages echoed from the strategy.
    while backtester.is_alive():
        message = raw_input("Enter next message: ")
        # Send messages to the strategy.
        print('Send message to strategy:  ' + message)
        mission_controller.send_to_strategy(topic, message)

        received = mission_controller.poll_from_strategy(topic=topic)
        for received_topic, received_message in received.items():
            print('Messages from strategy: {}: {}'.format(received_topic, received_message))

    backtester.join()
    mission_control.dispose()
    mission_controller.dispose()
    context.term()


if __name__ == '__main__':
    main()

```
