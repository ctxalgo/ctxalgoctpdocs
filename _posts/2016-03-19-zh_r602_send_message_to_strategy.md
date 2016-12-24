---
title: 向策略反复发送消息
layout: post
category: live
language: zh
---

本程序向一个正在运行的策略向策略反复发送消息。用于调试策略消息通讯。


```python
from optparse import OptionParser
import zmq
from time import sleep
from datetime import datetime, timedelta
from ctxalgolib.mission_control.mission_control import MissionController


def get_cmd_parser():
    parser = OptionParser()

    parser.add_option(
        '--chore', type='string', dest='chore', default=None,
        help='Zeromq queue address for the messages to send, without the port part. For example, tcp://127.0.0.1. '
             'If None, default IP will be used.')

    parser.add_option(
        '--strategy', type='string', dest='strategy',
        help='Name of the strategy to send messages to, in form of product.strategy.')

    parser.add_option(
        '--exit-time', type='string', dest='exit_time', default=None,
        help='Exit time of current message sending script, in form of YYYYMMDDHHMMSS. If None, in 24 hours.')

    parser.add_option(
        '--frequency', type='int', dest='frequency', default=60,
        help='Frequency in seconds of the message sending. For example, 60 means sending a message every 60 seconds.')

    parser.add_option(
        '--proxy', action='store_true', dest='proxy', default=False,
        help='Indicate to use zmq proxy.')

    return parser


def main():
    parser = get_cmd_parser()
    options, args = parser.parse_args()
    strategy = options.strategy
    if options.exit_time is None:
        exit_time = datetime.now() + timedelta(hours=24)
    else:
        exit_time = datetime.strptime(options.exit_time, '%Y%m%d%H%M%S')
    context = zmq.Context()
    mc = MissionController(context, strategy_name=strategy, proxy_ip=options.chore, proxy=options.proxy)

    # Sleep for a few seconds to reduce the problem of zmq slow-joiner.
    sleep(5)

    # Send messages.
    msg_id = 1
    while True:
        now = datetime.now()
        if now > exit_time:
            break
        mc.send_to_strategy(strategy, 'info: {}, {}'.format(msg_id, now))
        print('{} Sent message: {}'.format(now, msg_id))
        msg_id += 1
        sleep(options.frequency)

    mc.dispose()
    context.term()


if __name__ == '__main__':
    main()

```
