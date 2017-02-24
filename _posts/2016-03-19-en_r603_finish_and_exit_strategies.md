---
title: Send messages to terminate strategies.
layout: post
category: live
language: en
---

This script send messages to specified strategies to ask them to terminate themselves

```python
import os
import json
from optparse import OptionParser
import zmq
from time import sleep
from ctxalgolib.mission_control.mission_control import MissionController
from ctxalgolib.mission_control.message_constants import MissionControlMessages
from ctxalgoctp.ctp.position_utils import PositionUtils


def get_cmd_parser():
    parser = OptionParser()

    parser.add_option(
        '--chore', type='string', dest='chore', default=None,
        help='Zeromq queue address for the messages to send, without the port part. For example, tcp://127.0.0.1. '
             'If None, default IP will be used.')

    parser.add_option(
        '--strategies', type='string', dest='strategies',
        help="Names of the strategies to finish and exit, in form of comma-separated strategty names. "
             "If the value is a file path, then assume the file is in format strategy spec and load strategy "
             "names from that file.")

    return parser


def main():
    parser = get_cmd_parser()
    options, args = parser.parse_args()
    strategies = options.strategies
    if os.path.isfile(strategies):
        strategies = PositionUtils.load_strategy_map(strategies).keys()
    else:
        strategies = strategies.split(',')
    context = zmq.Context()
    mc = MissionController(context, strategy_name=strategies, proxy_ip=options.chore, proxy=True)

    # Sleep for a few seconds to reduce the problem of zmq slow-joiner.
    sleep(5)

    done = False
    for i in range(3):
        print (i)
        # Try to send messages for a couple of times to avoid slow joiner further.
        remaining_strategies = set(strategies)
        for s in remaining_strategies:
            print('Send finish_and_exit message to strategy {}.'.format(s))
            mc.send_to_strategy(s, MissionControlMessages.finish_and_exit)

        sleep(5)
        # Get replies from
        for w in range(5):
            replies = mc.poll_from_strategy(timeout=2)
            for replied_strategy, bodies in replies.items():
                for body in bodies:
                    if body.find(MissionControlMessages.finish_and_exit) >= 0:
                        remaining_strategies.remove(replied_strategy)
                        print('Received finish_and_exit reply from strategy {}.'.format(replied_strategy))
            if len(remaining_strategies) == 0:
                done = True
                break
        if done:
            break

    mc.dispose()
    context.term()


if __name__ == '__main__':
    main()

```
