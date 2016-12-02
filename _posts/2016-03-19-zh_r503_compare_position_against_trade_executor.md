---
title: 比较订单执行器中的仓位和与其关联的策略的仓位
layout: post
category: live
language: zh
---

本程序比较订单执行器中的仓位和与其关联的策略的仓位。


```python
import os
import sys
from ctxalgolib.output.texttable import Texttable
from optparse import OptionParser
from datetime import datetime
from ctxalgolib.trading_utils.trading_utils import AbstractTradingUtils
from ctxalgoctp.ctp.constants import Constants as C
from ctxalgoctp.ctp.trading_account import TradingAccount


def get_account_from_path(path):
    if os.path.isdir(path):
        path = os.path.join(path, C.account_file_name)
    return TradingAccount.from_json_file(None, path)


def save_position(positions, sid, actual_sid, direction):
    if sid in positions and actual_sid in positions[sid] and direction in positions[sid][actual_sid]:
        d = positions[sid][actual_sid][direction]
        return {'yesterday_volume': d['yesterday_volume'], 'today_volume': d['today_volume']}
    else:
        return {'yesterday_volume': 0, 'today_volume': 0}


def get_table(header, content, max_width=120):
    table = Texttable(max_width=max_width)
    table.set_deco(Texttable.HEADER)
    table.set_cols_align(['r'] * len(header))
    table.add_rows([header] + content)
    return table.draw()


def main():
    parser = OptionParser()
    parser.add_option(
        '--strategy', type='string', action='append', dest='strategies',
        help='Path to the base-folder or an account file of a strategy. You can provide multiple strategies by '
             'writing multiple --strategy sections. The first provided strategy will be named s1, the second s2, '
             'and so on.')

    parser.add_option(
        '--trade-executor', type='string', dest='trade_executor',
        help='Path to the base-folder or an account file of the trade executor associated with the strategies '
             'provided by --strategy sections.')

    parser.add_option(
        '--trading-day', type='string', dest='trading_day', default=None,
        help='Trading day in form of yyyymmdd, if not present, use current system time to decide trading day.')

    parser.add_option(
        '--table', action='store_true', dest='table', default=False,
        help='If present, print result in nicer ASCII table, otherwise, simple csv format.')

    options, args = parser.parse_args()

    if options.trading_day is None:
        trading_day = AbstractTradingUtils().trading_day_from_now(datetime.now())
    else:
        trading_day = datetime.strptime(options.trading_day ,'%Y%m%d').date()
    print_table = options.table

    # Load positions from trade executor and all individual strategies.
    trade_executor_positions = get_account_from_path(options.trade_executor).position_summary(trading_day=trading_day)
    positions = {'trader': trade_executor_positions}
    strategy_positions = {}
    strategies = []
    for id_, s in enumerate(options.strategies):
        name = 's{}'.format(id_+1)
        strategies.append(name)
        position = get_account_from_path(s).position_summary(trading_day=trading_day)
        strategy_positions[name] = position
        positions[name] = position

    # Get the full list of instruments from trade executor and all individual strategies.
    instruments = set([])
    for _, p in positions.items():
        for sid, p2 in p.items():
            for actual_sid, p3 in p2.items():
                instruments.add((sid, actual_sid))
    instruments = list(sorted(list(instruments)))
    # Compare position differences.
    differences = []
    for sid, actual_sid in instruments:
        for direction in ['long', 'short']:
            trade_volume = save_position(trade_executor_positions, sid, actual_sid, direction)
            strategy_volume = {'yesterday_volume': 0, 'today_volume': 0}
            sps = []
            for s in strategies:
                sv = save_position(strategy_positions[s], sid, actual_sid, direction)
                sps.append(sv)
                strategy_volume['yesterday_volume'] += sv['yesterday_volume']
                strategy_volume['today_volume'] += sv['today_volume']
            for day in ['yesterday_volume', 'today_volume']:
                if trade_volume[day] != strategy_volume[day]:
                    delta = strategy_volume[day] - trade_volume[day]
                    dname = day.split('_')[0]
                    row = [sid, actual_sid, direction, dname, trade_volume[day]] + [v[day] for v in sps] + [delta]
                    differences.append(row)

    # Print position differences, if any.
    if len(differences) > 0:
        if print_table:
            headers = ['sid', 'actual_sid', 'direction', 'date', 'trader'] + strategies + ['delta']
            print(get_table(headers, differences))
        else:
            for row in differences:
                print(u','.join([str(v) for v in row]))
        sys.exit(-1)
    else:
        sys.exit(0)


if __name__ == '__main__':
    main()



```
