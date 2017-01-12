---
title: Strategy log printer
layout: post
category: live
language: en
---

This example prints all the strategy logs received from a zmq queue.

```python
import zmq
from optparse import OptionParser
from ctxalgolib.rule_checking.configs import RuleCheckingConfigs
from ctxalgolib.scripts.gracefull_killer import GracefulKiller
from ctxalgolib.pub_subs.pub_sub_constants import PubSubConstants


def get_command_line_parser():
    parser = OptionParser()

    parser.add_option('--strategy-log-queue', type='string', dest='strategy_log_queue', default=None,
                      help='Queue to get strategy log.')
    return parser


def main():
    options, args = get_command_line_parser().parse_args()
    c = RuleCheckingConfigs()

    if options.strategy_log_queue is None:
        strategy_log_queues = [c.front_proxy_output_queue()]
    else:
        strategy_log_queues = options.strategy_log_queue.split(',')

    context = zmq.Context()

    poller = zmq.Poller()
    sockets = []
    for address in strategy_log_queues:
        subscriber = context.socket(zmq.SUB)
        subscriber.connect(address)
        subscriber.setsockopt(zmq.SUBSCRIBE, '')
        sockets.append(subscriber)
        poller.register(subscriber, zmq.POLLIN)

    g = GracefulKiller()
    while not g.kill_now:
        received = poller.poll(timeout=1)
        if len(received) > 0:
            for socket, _ in received:
                [message_topic, content] = socket.recv_multipart()
                parts = message_topic.split('#')
                header = parts[0] + '.' + parts[1] + '.'
                strategy_name = message_topic[len(header):]
                print('{}\t{}'.format(message_topic, content))

    for s in sockets:
        s.setsockopt(zmq.LINGER, PubSubConstants.linger)
        s.close()
    context.term()


if __name__ == '__main__':
    main()
```
