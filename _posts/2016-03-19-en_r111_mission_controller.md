---
title: Mission controller which communicate with strategies.
layout: post
category: live
language: en
---



```python
import zmq
from optparse import OptionParser
from ctxalgolib.rule_checking.configs import RuleCheckingConfigs
from ctxalgolib.mission_control.mission_control import MissionController


def get_command_line_parser():
    parser = OptionParser()

    parser.add_option('--strategies', type='string', dest='strategies', default='',
                      help='strategy_name,queue_from_strategy,queue_into_strategy[,strategy_name,queue_from_strategy,queue_into_strategy]* or strategy_name[,strategy_name]')
    parser.add_option('--proxy', action='store_true', dest='through_proxy', default=False,
                      help='If True, queues are connected through a proxy.')

    return parser


def main():
    options, args = get_command_line_parser().parse_args()
    context = zmq.Context()

    # Create a mission controller, which is outside of a strategy and sends and receives messages from a strategy.
    queues_from_strategy = {}
    queues_into_strategy = {}
    c = RuleCheckingConfigs()
    if options.through_proxy:
        for strategy in options.strategies.split(','):
            queues_from_strategy[strategy] = c.front_proxy_output_queue()
            queues_into_strategy[strategy] = c.back_proxy_input_queue()
    else:
        parts = options.strategies.split(',')
        for p in parts:
            info = p.strip().split('#')
            if len(info) == 3:
                strategy = info[0]
                from_queue = info[1]
                to_queue = info[2]
                queues_from_strategy[strategy] = from_queue
                queues_into_strategy[strategy] = to_queue
            else:
                assert len(info) == 1
                strategy = info[0]
                queues_from_strategy[strategy] = c.front_proxy_input_queue()
                queues_into_strategy[strategy] = c.back_proxy_output_queue()

    mission_controller = MissionController(
        context,
        queues_from_strategies=queues_from_strategy,
        queues_into_strategies=queues_into_strategy,
        proxy=options.through_proxy)

    while True:
        message = raw_input('Enter next message form of "strategy:message": ')
        # Send messages to the strategy.
        print('Send message to strategy:  ' + message)
        parts = message.split(':')
        if len(parts) >= 2:
            strategy, content = parts[0], ':'.join(parts[1:])
            mission_controller.send_to_strategy(strategy, content)

    mission_controller.dispose()
    context.term()


if __name__ == '__main__':
    main()

```
