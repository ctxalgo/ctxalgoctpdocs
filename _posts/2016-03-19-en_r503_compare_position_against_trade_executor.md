---
title: Compare positions between trade executor and associated individual strategies
layout: post
category: live
language: en
---

This script compares positions between a trade executor and its associated individual strategies.

```python
import os
import sys
from optparse import OptionParser
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils as Topics
from ctxalgoctp.ctp.constants import Constants as C
from ctxalgoctp.ctp.trading_account import TradingAccount
from ctxalgoctp.ctp.position_utils import PositionUtils


def get_account_from_path(path, trading_day=None):
    if os.path.isdir(path):
        if trading_day is None:
            path = os.path.join(path, C.account_file_name)
        else:
            for f in reversed(sorted(os.listdir(os.path.join(path, C.account_backup_folder_name)))):
                parts = f.split('_')
                td = parts[1]
                if td == trading_day.strftime('%Y%m%d'):
                    path = os.path.join(path, C.account_backup_folder_name, f)
                    break
    return TradingAccount.from_json_file(None, path)


def save_position(positions, sid, actual_sid, direction):
    if sid in positions and actual_sid in positions[sid] and direction in positions[sid][actual_sid]:
        d = positions[sid][actual_sid][direction]
        return {'yesterday_volume': d['yesterday_volume'], 'today_volume': d['today_volume']}
    else:
        return {'yesterday_volume': 0, 'today_volume': 0}


def main():
    parser = OptionParser()
    parser.add_option(
        '--strategy', type='string', action='append', dest='strategies',
        help='Path to the base-folder or an account file of a strategy. You can provide multiple strategies by '
             'writing multiple --strategy sections. The first provided strategy will be named s1, the second s2, '
             'and so on. You can use one or more --strategy to specify individual strategies, or you can use '
             '--log-folder and --strategy-map to specify those strategies.')

    parser.add_option(
        '--trade-executor', type='string', dest='trade_executor', default=None,
        help='Path to the base-folder or an account file of the trade executor associated with the strategies '
             'provided by --strategy sections.')

    parser.add_option(
        '--trading-day', type='string', dest='trading_day', default=None,
        help='Trading day in form of yyyymmdd, if not present, use current system time to decide trading day.')

    parser.add_option(
        '--table', action='store_true', dest='table', default=False,
        help='If present, print result in nicer ASCII table, otherwise, simple csv format.')

    parser.add_option(
        '--strategy-log-folder', type='string', dest='strategy_log_folder', default=None,
        help='Path to a folder that contains base folders of all the relevant strategies and their corresponding '
             'trade executors. ')

    parser.add_option(
        '--strategy-map', type='string', dest='strategy_map', default=None,
        help='Path to a strategy map file. If there is no --strategy provided, current script will use --strategy-map '
             'to find the strategy names and use --log-folder to find the folder that contains base folders of those '
             'strategies and the corresponding trade executor. The file is in csv format, with row structure: '
             'CTX strategy name, expected position strategy name, entry time, (optional) scaling factor. '
             'If the fourth column, the scaling factor column, is missing, default_scaling_factor is used.\n'
             'For example:\n'
             'simulation.sa_trend_1445,sa_trend,14:45,0.1\n'
             'simulation.good_morning_095500,good_morning,9:55,0.1\n')

    parser.add_option(
        '--trade-executor-name', type='string', dest='trade_executor_name', default=None,
        help='The trade executor strategy name. This is used when --trade-executor is not given. Together with '
             '--strategy-log-folder, it defines the base folder of the trade executor. It can be in form of '
             'either product.strategy, or just strategy. If the former, the product part must match the product '
             'defined in --strategy-map.')

    options, args = parser.parse_args()
    trading_day = PositionUtils.trading_day_from_now(options.trading_day)
    print_table = options.table

    positions = {}
    strategy_positions = {}
    strategies = []
    strategy_map = {}
    product = None
    if options.strategies is not None and len(options.strategies) > 0:
        # Individual strategies are given by --strategy command line.
        for id_, s in enumerate(options.strategies):
            name = 's{}'.format(id_+1)
            strategy_map[name] = s
            strategies.append(name)
            position = get_account_from_path(s, trading_day=trading_day).position_summary(trading_day=trading_day)
            strategy_positions[name] = position
            positions[name] = position
    else:
        # Individual strategies are given by strategy map file.
        assert options.strategy_map is not None
        assert options.strategy_log_folder is not None
        for id_, s_name in enumerate(PositionUtils.load_strategy_map(options.strategy_map).keys()):
            name = 's{}'.format(id_ + 1)
            base_folder = os.path.join(options.strategy_log_folder, s_name)
            product = s_name.split(Topics.strategy_signature_separator())[0]
            strategy_map[name] = base_folder

    for name, base_folder in strategy_map.items():
        strategies.append(name)
        position = get_account_from_path(base_folder, trading_day=trading_day).position_summary(trading_day=trading_day)
        strategy_positions[name] = position
        positions[name] = position

    # Load positions from trade executor and all individual strategies.
    if options.trade_executor is not None and os.path.isdir(options.trade_executor):
        trade_executor_positions = get_account_from_path(options.trade_executor, trading_day=trading_day).position_summary(trading_day=trading_day)
        positions['trader'] = trade_executor_positions
        trader_base_folder = options.trade_executor
    else:
        assert options.strategy_map is not None
        assert options.strategy_log_folder is not None
        assert product is not None
        assert options.trade_executor_name is not None
        trader = options.trade_executor_name
        sep_pos = trader.find(Topics.strategy_signature_separator())
        if sep_pos >= 0:
            p = trader[:sep_pos]
            assert p == product
            trader = trader[sep_pos+1:]
        trader_signature = Topics.strategy_signature(product, trader)
        trader_base_folder = os.path.join(options.strategy_log_folder, trader_signature)
        trade_executor_positions = get_account_from_path(trader_base_folder, trading_day=trading_day).position_summary(trading_day=trading_day)
        positions['trader'] = trade_executor_positions

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
        print('Strategies')
        for s in sorted(strategy_map.keys()):
            print('{}: {}'.format(s, strategy_map[s]))
        print('trader: {}'.format(trader_base_folder))
        print('')

        print('Position differences')
        if print_table:
            headers = ['sid', 'actual_sid', 'direction', 'date', 'trader'] + strategies + ['delta']
            print(PositionUtils.get_table(headers, differences))
        else:
            for row in differences:
                print(u','.join([str(v) for v in row]))
        sys.exit(-1)
    else:
        sys.exit(0)


if __name__ == '__main__':
    main()



```
