---
title: 将策略中收到的交易数据发送到zeromq队列中
layout: post
category: live
language: zh
---

本程序展示了入股将策略里收到的tick数据和K线数据发送到zeromq队列中去。本程序同时展示了在不同的线程中进行zeromq队列
发送操作的方式。


```python
import zmq
from threading import Thread
from Queue import Queue, Empty
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils as Topics
from ctxalgoctp.ctp.live_strategy_utils import *


class FakeMarketBroker(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, tick_cache, bar_cache,
                 periods=None, description=None, logger=None):
        """
        :param tick_cache: Queue, the Python Queue to store ticks that need to be sent into zmq.
        :param bar_cache: Queue, the Python Queue to store bars that need to be send into zmq.
        """
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_should_terminate_after_market_close(True)
        self.tick_cache = tick_cache
        self.bar_cache = bar_cache

    def on_tick(self, instrument_id, tick):
        topic = Topics.get_tick_filter('future', instrument_id)
        self.tick_cache.put((topic, tick))

    def on_bar(self, instrument_id, bars, tick):
        # TODO: Go to ctxalgocore.market_data.market_data_broker, line 996, the _on_bar_generated_using_last method
        # TODO: to see how to consume the `bars` data and send them into zmq.
        pass


class ZmqDataSenderThread(Thread):
    """
    Class to send messages into a zmq queue.
    """
    def __init__(self, cache, socket_address, proxy, context):
        """
        :param cache: Queue, the Python Queue object containing data to be sent to zmq.
        :param socket_address: string, zmq socket address.
        :param proxy: bool, if True, the socket is going through a proxy.
        :param context: zmq Context.
        """
        assert socket_address is not None
        Thread.__init__(self)
        self.socket_address = socket_address
        self.should_exit = False
        self.context = context
        self.proxy = proxy
        self._queue = cache

    def set_should_exit(self, v):
        # Assume that there is only one thread which calls this method, so we don't do thread synchronization here.
        self.should_exit = v

    def run(self):
        socket = self.context.socket(zmq.PUB)
        if self.proxy:
            socket.connect(self.socket_address)
        else:
            socket.bind(self.socket_address)

        while not self.should_exit:
            try:
                topic, content = self._queue.get(timeout=0.001)
                socket.send_multipart([topic, content])
            except Empty:
                pass
            except Exception:
                raise

        socket.setsockopt(zmq.LINGER, 0)
        socket.close()


def main():
    context =zmq.Context()

    # Setup command line options.
    cmd_options = '--account simnow_future3 --name test.fake_market_broker --instruments cu1610,SR611'
    parser = get_command_line_parser(strategy_class=FakeMarketBroker, cmd_options=cmd_options, context=context)
    parser.add_option(
        '--bar-queue', type='string', dest='bar_queue', default='tcp://127.0.0.1:6557',
        help="The zmq queue address to send bar data.")
    parser.add_option(
        '--tick-queue', type='string', dest='bar_queue', default='tcp://127.0.0.1:6556',
        help="The zmq queue address to tick bar data.")
    parser.add_option(
        '--proxy', action='store_true', dest='proxy', default=False,
        help='If True, queues are connected through a proxy.')

    options = parser.parse()
    assert options.bar_queue is not None
    assert options.tick_queue is not None

    # Create queue for bar data.
    bar_cache = Queue()
    bar_sender = ZmqDataSenderThread(bar_cache, options.bar_queue, options.proxy, context)
    bar_sender.start()

    # Create queue for tick data.
    tick_cache = Queue()
    tick_sender = ZmqDataSenderThread(tick_cache, options.tick_queue, options.proxy, context)
    tick_sender.start()

    # Run strategy.
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'periods': [Periodicity.ONE_MINUTE],
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger(),
        'tick_cache': tick_cache,
        'bar_cache': bar_cache
    }
    run_live(FakeMarketBroker, config, options)

    # Dispose queues and threads.
    bar_sender.set_should_exit(True)
    tick_sender.set_should_exit(True)
    bar_sender.join()
    tick_sender.join()

    context.term()


if __name__ == '__main__':
    main()

```
