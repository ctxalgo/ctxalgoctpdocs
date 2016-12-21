---
title: Send messages to strategy repeatedly
layout: post
category: live
language: en
---

This script sends messages to a running strategy repeatedly. Used for debugging message communication.

```python
from optparse import OptionParser
import zmq
from time import sleep
from datetime import datetime
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
        '--exit-time', type='string', dest='exit_time',
        help='Exit time of current message sending script, in form of YYYYMMDDHHMMSS.')

    parser.add_option(
        '--frequency', type='int', dest='frequency', default=60,
        help='Frequency in seconds of the message sending. For example, 60 means sending a message every 60 seconds.')

    return parser


def main():
    parser = get_cmd_parser()
    options, args = parser.parse_args()
    strategy = options.strategy
    exit_time = datetime.strptime(options.exit_time, '%Y%m%d%H%M%S')

    context = zmq.Context()
    mc = MissionController(context, proxy=True, strategy_name=strategy, proxy_ip=options.chore)

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
