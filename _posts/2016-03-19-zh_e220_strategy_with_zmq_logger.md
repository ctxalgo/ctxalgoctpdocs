---
title: 给策略连接一个基于zeromq的日志输出
layout: post
category: root
language: zh
---

本示例展示如何给交易策略连接一个基于zeromq的日志输出。一个交易策略可以连接多个日志输出。我们之前已经使用过了基于文件的
日志输出，这个输出会把来自策略的消息写入一个文件。我们也使用过了命令行日志输出，这个输出把来自策略的消息打印到命令行。

基于zeromq的日志输出会把来自策略的消息发送到指定的zeromq队列中。这个输出有以下一些用途：

1. 在队列的另一端可以连接一个图形界面，来显示策略运行的实时情况。
2. 在队列的另一端可以连接一个规则检查器，用来检查策略的下单是否满足程序化接入的规则，比如下单速度不能过快等。


```python
import threading
import zmq
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import *
```

要给策略连接一个基于zeromq的日志输出，只需要设置策略类`__init__`方法中的`logger`参数即可。这里，我们使用
`StartegyUtils.get_logger`来返回一个包含zeromq日志输出的对象。然后，我们在一个单独的线程中启动策略回测，并且在
主线程中监听来自zeromq队列的策略的消息，并把这些消息打印出来。


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
    queue = 'tcp://127.0.0.1:3000'
    context = zmq.Context()

    # Create a subscriber which gets message from the strategy.
    subscriber = context.socket(zmq.SUB)
    subscriber.connect(queue)
    subscriber.setsockopt(zmq.SUBSCRIBE, '')

    backtester = StrategyBacktestingThread(queue, context)
    backtester.start()

    # Output received strategy messages until the backtesting finishes.
    c = 0
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
