---
title: Use zeromq-based data producer
layout: post
category: live
language: en
---

This example shows how to use a zeromq-based data producer. By using the zeromq-based data producer, the strategy
won't need to connect to CTP market server anymore. Use `--data-producer=IP` to specify that zeromq data producer.


```python
import zmq
from ctxalgoctp.ctp.live.r100_connect_to_ctp import JustConnect
from ctxalgoctp.ctp.live_strategy_utils import *


def main():
    context = zmq.Context()
    cmd_options = '--account simnow_future4 name test.s1 --data-producer tcp://139.196.203.113 ' \
                  '--instruments cu1610,SR611'
    parser = get_command_line_parser(JustConnect, context=context, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'periods': [Periodicity.ONE_MINUTE, Periodicity.FIVE_MINUTE],
        'parameters': {
            'exit_in_time': 600
        },
        'logger': options.get_logger()
    }
    run_live(JustConnect, config, options)
    context.term()

if __name__ == '__main__':
    main()

```
