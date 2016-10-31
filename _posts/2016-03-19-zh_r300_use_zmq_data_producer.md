---
title: 连接基于zeromq的行情数据产生器
layout: post
category: live
language: zh
---

本策略展示如何给策略连接一个基于zeromq的行情数据产生器。在使用了这个数据产生器后，策略将不需要直接连接到CTP的行情服务器
上。通过`--data-producer=IP`来指定使用zeromq行情产生器。


```python
import zmq
from ctxalgoctp.ctp.live.r100_connect_to_ctp import JustConnect
from ctxalgoctp.ctp.live_strategy_utils import *


def main():
    context = zmq.Context()
    cmd_options = '--account simnow_future4 --name test.s1 --data-producer tcp://139.196.203.113 ' \
                  '--instruments cu1611,SR701'
    parser = get_command_line_parser(JustConnect, context=context, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'periods': [Periodicity.ONE_MINUTE, Periodicity.FIVE_MINUTE],
        'parameters': {
            'exit_in_second': 600
        },
        'logger': options.get_logger()
    }
    run_live(JustConnect, config, options)
    context.term()

if __name__ == '__main__':
    main()

```
