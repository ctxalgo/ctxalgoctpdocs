---
title: Strategy with zeromq logger
layout: post
category: en
---

This example demonstrates how to attach a zeromq logger to a strategy. A strategy can have different loggers,
we have used file-based logger, which stores all messages from a strategy in a file, and console logger,
which outputs all messages to the console.
The zeromq-based logger will send all the messages to a zeromq queue. Typical use of such a logger includes:
1. Have a GUI to visualize the progress of the strategy.
2. A rule checker will check orders from the strategy to see if they satisfy algo-trading regulations.

```python
import threading
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import *
```

To Attach a zeromq logger to a strategy, specify the `logger` parameter at the strategy's `__init__` method. Here,
we use the `StrategyUtils.get_logger` method to create such as logger. Then, we run the backtesting in a separate
thread, and in the main thread, we use a zeromq subscriber to listen to and print the messages from the strategy.

```python
class StrategyBacktestingThread(threading.Thread):
    """
    This thread runs backtesting on TrendFollowingStrategy.
    """
    def __init__(self, queue, context):
        """
        :param queue: string, the zeromq queue to which strategy logs are sent.
        :param context: zeromq context.
        """
        threading.Thread.__init__(self)
        self.queue = queue
        self.context = context

    def run(self):
        start_date = '2014-01-01'  # Backtesting start date.
        end_date = '2014-12-31'  # Backtesting end date.

        base_folder = os.path.join(os.environ['CTXALGO_TEST'], 'strategies', TrendFollowingStrategy.__name__)
        config = {
            'instrument_ids': ['IF99'],
            'strategy_period': Periodicity.FIFTEEN_MINUTE,
            'parameters': {
                'fast_ma_period': 8,
                'slow_ma_period': 15,
            },
            'base_folder': base_folder,
            # Use queue_address and queue_context to create a zeromq logger.
            # The logger will send all messages from strategy to the zeromq queue.
            'logger': StrategyUtils.get_logger(
                base_folder, has_console=False, queue_address=self.queue, queue_context=self.context)
        }
        backtest(TrendFollowingStrategy, config, start_date, end_date)


def main():
    queue = 'tcp://127.0.0.1:5555'
    context = zmq.Context()

    # Create a subscriber which gets message from the strategy.
    subscriber = context.socket(zmq.SUB)
    subscriber.connect(queue)
    subscriber.setsockopt(zmq.SUBSCRIBE, '')

    backtester = StrategyBacktestingThread(queue, context)
    backtester.start()

    # Output received strategy messages until the backtesting finishes.
    while backtester.is_alive():
        try:
            [topic, content] = subscriber.recv_multipart(flags=zmq.NOBLOCK)
            print('{}\t{}'.format(topic, content))
        except Exception as e:
            pass
    backtester.join()

    subscriber.setsockopt(zmq.LINGER, 0)
    subscriber.close()
    context.term()


if __name__ == '__main__':
    main()

```
