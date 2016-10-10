---
title: Setup historical ohlc bars in live trading.
layout: post
category: live
language: en
---

When a strategy needs historical ohlc bars, you need to setup the bar requirement so historical ohlcs will be
downloaded and prepared for you. Use the command line option --historical-ohlc-length to specify the maximum
number of bars you need for all kinds of ohlcs. And use --ohlc-fields to indicate which fields you need, besides
the usual open, high, low, close, volumes.


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
