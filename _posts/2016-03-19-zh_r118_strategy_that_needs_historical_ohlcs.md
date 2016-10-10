---
title: 在实盘策略中配置历史K线
layout: post
category: live
language: zh
---

如果策略需要历史K线，你需要使用--historical-ohlc-length来指定这些K线的长度，同时使用--ohlc-fields来指定这些K线需要包括
哪些字段。


```python
from ctxalgoctp.ctp.live_strategy_utils import *
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy


def main():
    # Use --historical-ohlc-length to specify the need for historical ohlcs.
    # Use --ohlc-fields to specify the fields that are needed in ohlc bars.
    cmd_options = '--account simnow_future4 --name test.s1 --instruments cu1611 ' \
                  '--historical-ohlc-length 200 --ohlc-fields dominants,profits,open_interest,book0'
    parser = get_command_line_parser(TrendFollowingStrategy, cmd_options=cmd_options)
    options = parser.parse()
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'logger': options.get_logger()
    }
    run_live(TrendFollowingStrategy, config, options=options)


if __name__ == '__main__':
    main()

```
