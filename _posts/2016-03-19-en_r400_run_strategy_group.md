---
title: Run strategies in a group
layout: post
category: live
language: en
---

This script shows how to run multiple strategies in the same group. 

All strategies in the same group will be run in the same process.


```python
from ctxalgoctp.ctp.live_strategy_utils import *
from ctxalgoctp.ctp.live.r100_connect_to_ctp import JustConnect


def main():
    cfgs = []
    opts = []
    strategy_count = 2

    # Run two strategies in a group. Each strategy has a different name, and checks a different
    for name, sid in zip(['test.s'+str(i+1) for i in range(strategy_count)], ['cu1705', 'i1709']*strategy_count):
        cmd_options = '--account simnow_future --name {} --instruments {}'.format(name, sid)
        parser = get_command_line_parser(strategy_class=JustConnect, cmd_options=cmd_options)
        parser.add_option(
            '--exit', type='int', dest='exit', default=100,
            help='Exit strategy in the given seconds. ')
        parser.add_option(
            '--no-trading', action='store_false', dest='trading_enabled', default=True,
            help='Should trading be enabled.')
        parser.add_option(
            '--no-market', action='store_false', dest='market_enabled', default=True,
            help='Should market be enabled.')

        options = parser.parse()
        config = {
            'instrument_ids': options.get_instruments(),
            'periods': [Periodicity.ONE_MINUTE],
            'parameters': {
                'exit_in_second': options.exit,
                'trading_enabled': options.trading_enabled,
                'market_enabled': options.market_enabled,
                'local_bookkeep': options.local_bookkeep,
            },
            'description': options.get_description(),
            'logger': options.get_logger()
            }
        cfgs.append(config)
        opts.append(options)

    run_live([JustConnect] * strategy_count, cfgs, opts)


if __name__ == '__main__':
    main()

```
