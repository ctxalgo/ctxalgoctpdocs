---
title: 从交易账户和期望仓位来重新分配策略中的仓位
layout: post
category: live
language: zh
---

从交易账户和期望仓位来重新分配策略中的仓位


```python
import zmq
import sys
import math
from optparse import OptionParser
from ctxalgolib.rule_checking.rule_checker_message_sender import RuleCheckerMessageSender
from ctxalgolib.rule_checking.error_codes import ErrorCodes
from ctxalgoctp.ctp.live_strategy_utils import *
from ctxalgoctp.ctp.constants import Constants as C
from ctxalgoctp.ctp.position_utils import PositionUtils
from ctxalgoctp.ctp.position_rebalancer import PositionRebalancer
from ctxalgoctp.ctp.trading_account_loaders.local_trading_account_loader import LocalTradingAccountLoader


def get_cmd_parser():
    parser = OptionParser()

    parser.add_option(
        '--account', type='string', dest='account', default=None,
        help='Path to a trading account config, the config stores a CTP trading account connection information, such as'
             'trading server, user, password, broker id. Or a simulation account name, such simnow_future, '
             'defined in CTX.')

    parser.add_option(
        '--account-position-file', type='string', dest='account_position_file', default=None,
        help='Path to strategy position file. If this is given, --account will be ignored and current script will load '
             'trading account positions from the given file. ')

    parser.add_option(
        '--security-company', type='string', dest='security_company', default=None,
        help='Security company, use default None is ok.')

    parser.add_option(
        '--trading-day', type='string', dest='trading_day', default=None,
        help='If --position-file is given, use this trading day in form of yyyymmdd. If None, use current system time, '
             'to decide a proper trading day.')

    parser.add_option(
        '--strategy', type='string', action='append', dest='strategies', default=[],
        help='Path to base folder of a strategy of account file of a strategy. To specify multiple strategies, '
             'add multiple --strategy sections.')

    parser.add_option(
        '--trade-executor', type='string', dest='trade_executor', default=None,
        help='Path to the base folder, or account file of the trade executor. '
             'If not given, no trade executor will be considered.')

    parser.add_option(
        '--strategy-map', type='string', dest='strategy_map',
        help='Path to strategy map file. This file defines the mapping from CTX strategy to that in an expected '
             'position file. The file is in csv format, with the following row structure: '
             'CTX strategy name, expected position strategy name, entry time, (optional) scaling factor. '
             'If the fourth column, the scaling factor column, is missing, default_scaling_factor is used.\n'
             'For example:\n'
             'simulation.sa_trend_1445,sa_trend,14:45,0.1\n'
             'simulation.good_morning_095500,good_morning,9:55,0.1\n')

    parser.add_option(
        '--expected-position', type='string', dest='expected_positions', default=None,
        help='Path to expected position file.')

    parser.add_option(
        '--external-position', type='string', dest='external_position', default=None,
        help='Path to external position file.')

    parser.add_option(
        '--port-weights', type='string', dest='port_weights', default=None,
        help='Path to port weights file.')

    parser.add_option(
        '--scaling-factor', type='float', dest='scaling_factor', default=0.1,
        help='Give default scaling factor for strategies, this value will be used if there is no scaling factor '
             'defined for a strategy in the strategy map file.')

    parser.add_option(
        '--overwrite', action='store_true', dest='overwrite', default=False,
        help='If present, write the synced position into account.txt in source strategy base folder, '
             'otherwise, write into a new file in the base folder.')

    parser.add_option(
        '--table', action='store_true', dest='table', default=False,
        help='If present, print position difference in nicer ASCII table, otherwise, simple csv format.')

    parser.add_option(
        '--view-only', action='store_true', dest='view_only', default=False,
        help='If present, only display position differences among trading account, strategies and trade executor, '
             'if given, and do not perform position re-balance.')

    parser.add_option(
        '--strategy-log-folder', type='string', dest='strategy_log_folder', default=None,
        help='Path to a folder that contains base folders of all the relevant strategies and their corresponding '
             'trade executors. ')

    parser.add_option(
        '--trade-executor-name', type='string', dest='trade_executor_name', default=None,
        help='The trade executor strategy name. This is used when --trade-executor is not given. Together with '
             '--strategy-log-folder, it defines the base folder of the trade executor. It can be in form of '
             'either product.strategy, or just strategy. If the former, the product part must match the product '
             'defined in --strategy-map.')

    parser.add_option('--rule-checker', type='string', dest='rule_checker', default=None,
                      help="Zeromq address, with port to rule checker input.")

    parser.add_option(
        '--only-trading-day', action='store_true', dest='only_trading_day', default=False,
        help='If present, only perform checking/position re-balancing on trading day. If not on a trading day, '
             'do nothing and terminate.')
    return parser


def get_account_info(options):
    if options.account_position_file is None:
        # FIXME: handle the case when retrieving account position fails.
        assert options.account is not None
        retrieve = PositionUtils.get_account_positions_through_strategy
        account_positions, trading_day, future_info_fac, position_prices = retrieve(
            options.account, None)
    else:
        retrieve = PositionUtils.get_account_positions_through_file
        account_positions, trading_day, future_info_fac, position_prices = retrieve(
            options.account_position_file, options.security_company, PositionUtils.trading_day_from_now(options.trading_day))
    return account_positions, trading_day, future_info_fac, position_prices


def compare_position_differences(account_position, strategy_positions, trader_position, strategy_short_names,
                                 table=False):
    strategy_sum = {}
    for s, p in strategy_positions.items():
        TradingAccount.add_compact_position_summary(strategy_sum, p)

    pos_equal = TradingAccount.is_compact_position_summary_equal(account_position, strategy_sum)
    if pos_equal and trader_position is not None:
        pos_equal = TradingAccount.is_compact_position_summary_equal(strategy_sum, trader_position)

    if pos_equal:
        return 0, ''

    columns = ['actual sid', 'direction', 'date', 'account'] + sorted(strategy_short_names.keys())
    if trader_position is not None:
        columns += ['trader']

    columns += ['account delta']
    if trader_position is not None:
        columns += ['trader delta']

    rows = []
    for sid in sorted(account_position.keys()):
        for d in ['long', 'short']:
            proceed = d in account_position[sid] and account_position[sid][d]
            if not proceed:
                proceed = any([d in strategy_positions.values()[sid]])
            if not proceed and trader_position is not None:
                proceed = d in trader_position[sid]
            if proceed:
                for day, dn in zip(['yesterday_volume', 'today_volume'], ['historical', 'today']):
                    row = []
                    row.append(sid)
                    row.append(d)
                    row.append(dn)
                    row.append(account_position[sid][d][day])
                    strategy_sum = 0
                    for short_name in sorted(strategy_short_names.keys()):
                        strategy_pos = strategy_positions[strategy_short_names[short_name]]
                        v = strategy_pos[sid][d][day]
                        strategy_sum += v
                        row.append(v)
                    if trader_position is not None:
                        row.append(trader_position[sid][d][day])
                    row.append(account_position[sid][d][day] - strategy_sum)
                    if trader_position is not None:
                        row.append(account_position[sid][d][day] - trader_position[sid][d][day])
                    if trader_position is None:
                        if row[-1] != 0:
                            rows.append(row)
                    else:
                        if row[-1] != 0 or row[-2] != 0:
                            rows.append(row)

    if len(rows) > 0:
        output = 'Strategies:\n'
        for s in sorted(strategy_short_names.keys()):
            output += '{}: {}\n'.format(s, strategy_short_names[s])
        output += '\nPosition differences\n'

        if table:
            output += PositionUtils.get_table(columns, rows) + '\n'
        else:
            output += ','.join(columns) + '\n'
            for row in rows:
                output += ','.join([str(c) for c in row]) + '\n'
        output += '\n'
        print(output)
        return -1, output


def position_distance(strategy_positions, expected_positions):
    result = 0.0
    instrument_count = 0
    for s, pos in strategy_positions.items():
        instrument_count = float(len(pos.keys()))
        for sid, pos2 in pos.items():
            for direction, pos3 in pos2.items():
                d = float(expected_positions[s][sid][direction] - (pos3['yesterday_volume'] + pos3['today_volume']))
                d *= d
                result += d

    if result > 0:
        result = math.sqrt(result) / instrument_count
    return result


def send_exception_to_rule_checker(source, title, content, rule_checker, error_code):
    if rule_checker is not None:
        context = zmq.Context()
        p = RuleCheckerMessageSender(context, address=rule_checker, sleep_second=5)
        p.send_general_exception(source, title, content, error_code=error_code)
        p.dispose()
        context.term()


def main():
    options, args = get_cmd_parser().parse_args()

    if options.only_trading_day:
        now = datetime.now().date()
        if not TradingDays().is_trading_day(now):
            print('{} is not a trading day, and --only-on-trading-day is present, do nothing and terminate.')
            sys.exit(0)

    try:
        # Retrieve positions from the given trading account.
        # account_positions are compact positions.
        account_positions, trading_day, future_info_fac, position_prices = get_account_info(options)

        # Load external positions.
        external_positions = PositionUtils.load_external_positions(options.external_position)
        external_positions = TradingAccount.compact_position_summary(external_positions)

        # Subtract external positions from account positions, the left over part is the positions that we re-balance.
        TradingAccount.subtract_compact_position_summary(account_positions, external_positions)
        # instruments is the set of actual instrument ids that we need to re-balance.
        instruments = set(account_positions.keys())

        # Load port weights.
        port_weights = PositionUtils.read_port_weights(options.port_weights)

        # Load strategy map. Strategy map connects the CTX strategy positions to strategies defined in
        # the expected positions file.
        product = None
        strategy_map = PositionUtils.load_strategy_map(
            options.strategy_map, default_scaling_factor=options.scaling_factor)
        if len(strategy_map) > 0:
            product = strategy_map.keys()[0].split(Topics.strategy_signature_separator())[0]

        # Load expected positions. Expected positions are the positions that the strategies are expected to have.
        if options.expected_positions is None:
            expected_position = {}
        else:
            expected_position = PositionUtils.parse_expected_positions(options.expected_positions)

        # Retrieve positions from the given strategies.
        # Type: dict{string: dict}, keys are base folders for strategies, values are information about those strategies.
        strategy_infos = {}
        strategy_short_names = {}
        if options.strategies is not None and len(options.strategies) > 0:
            for id_, s in enumerate(options.strategies):
                short_name = 's{}'.format(id_ + 1)
                info = PositionUtils.get_account_from_strategy(s, trading_day)
                base_folder = info['base_folder']
                strategy_infos[base_folder] = info
                strategy_short_names[short_name] = base_folder

        else:
            assert options.strategy_log_folder is not None
            for id_, s_name in enumerate(strategy_map.keys()):
                short_name = 's{}'.format(id_ + 1)
                base_folder = os.path.join(options.strategy_log_folder, s_name)
                info = PositionUtils.get_account_from_strategy(base_folder, trading_day)
                base_folder = info['base_folder']
                strategy_infos[base_folder] = info
                strategy_short_names[short_name] = base_folder
        assert all([v is not None for v in strategy_infos.values()])

        # Retrieve position from trade executor if given.
        trade_executor_base_folder = options.trade_executor
        trade_executor_info = None
        compact_trade_executor_positions = None
        if trade_executor_base_folder is None or not os.path.exists(trade_executor_base_folder):
            trader = options.trade_executor_name
            if options.strategy_log_folder is not None and trader is not None and product is not None:
                sep_pos = trader.find(Topics.strategy_signature_separator())
                if sep_pos >= 0:
                    p = trader[:sep_pos]
                    assert p == product
                    trader = trader[sep_pos + 1:]
                trader_signature = Topics.strategy_signature(product, trader)
                trade_executor_base_folder = os.path.join(options.strategy_log_folder, trader_signature)

        if trade_executor_base_folder is None or not os.path.exists(trade_executor_base_folder):
            trade_executor_info = PositionUtils.get_account_from_strategy(trade_executor_base_folder, trading_day)
            trade_executor_base_folder = trade_executor_info['base_folder']
            compact_trade_executor_positions = PositionUtils.padded_positions(
                instruments, TradingAccount.compact_position_summary(trade_executor_info['position_summary']))

        # For each strategy, find its expected positions (with port weight and scaling factor adjustment).
        strategy_positions = {}
        expected_positions = {}
        for base_folder, data in strategy_infos.items():
            strategy_signature = data['signature']
            assert strategy_signature in strategy_map, '{} not in strategy map.'.format(strategy_signature)
            entry_time = strategy_map[strategy_signature]['entry_time']
            expected_name = strategy_map[strategy_signature]['expected_name']
            scaling_factor = strategy_map[strategy_signature]['scaling_factor']

            if options.expected_positions is None:
                expected_positions[base_folder] = None
            else:
                assert expected_name in expected_position, '{} not in expected_positions.'.format(expected_name)
                assert entry_time in expected_position[expected_name], '{} not in {}'.format(entry_time, expected_name)
                port_weight = port_weights[expected_name] if expected_name in port_weights else 1.0
                expected_positions[base_folder] = PositionUtils.padded_expected_positions(
                    instruments, PositionUtils.adjusted_positions(
                        expected_position[expected_name][entry_time], port_weight, scaling_factor))

            # Pad strategy positions and expected positions with instruments in account.
            strategy_positions[base_folder] = PositionUtils.padded_positions(
                instruments, TradingAccount.compact_position_summary(data['position_summary']))

            if expected_positions[base_folder] is not None:
                assert set(strategy_positions[base_folder].keys()) == set(expected_positions[base_folder].keys())
            assert set(strategy_positions[base_folder].keys()) == instruments

        pos_diff, output = compare_position_differences(
            account_positions, strategy_positions, compact_trade_executor_positions, strategy_short_names,
            table=options.table)

        if pos_diff == 0:
            print('There is no position difference, do nothing.')
            sys.exit(0)

        if options.view_only:
            print('Position is not re-balanced because script is run with --view-only mode.')
            if pos_diff == 0:
                sys.exit(0)
            else:
                if options.rule_checker is not None:
                    # Send positions difference to rule checker.
                    source = 'Strategy position checker'
                    title = 'Strategy positions in product {} mismatch against trader or account.'.format(product)
                    send_exception_to_rule_checker(
                        source, title, output, options.rule_checker, error_code=ErrorCodes.s0002)
                sys.exit(-1)

        print('Position distance before re-balancing: {}'.format(
            position_distance(strategy_positions, expected_positions)))

        # Re-balance positions.
        new_positions = PositionRebalancer.rebalance_strategies(
            account_positions, strategy_positions, expected_positions)

        print('Position distance after re-balancing: {}'.format(position_distance(new_positions, expected_positions)))
        print('')

        # Construct strategies' positions from rebalanced positions.
        now = datetime.now()
        for base_folder, strategy_info in strategy_infos.items():
            account = strategy_info['account']
            LocalTradingAccountLoader(future_info_fac).change_positions(
                account, new_positions[base_folder], now, strategy_info['dominant_provider'],
                position_prices, trading_day_as_now=trading_day)
            PositionUtils.save_account(account, strategy_info['uuid'], base_folder, trading_day, options.overwrite)

        # Sync position for trade executor if provided.
        if trade_executor_info is not None:
            base_folder = trade_executor_base_folder
            account = trade_executor_info['account']
            LocalTradingAccountLoader(future_info_fac).change_positions(
                account, account_positions, now, trade_executor_info['dominant_provider'],
                position_prices, trading_day_as_now=trading_day)
            PositionUtils.save_account(
                account, trade_executor_info['uuid'], base_folder, trading_day, options.overwrite)

        print('Position re-balanced.')
        if not options.overwrite:
            print('Note: the synced position file is stored as {} in various base folders, you need to run current '
                  'script with the --overwrite command line option.'.format(C.synced_account_file_name))

    except Exception as e:
        content = traceback.format_exc()
        print(content)
        if options.rule_checker is not None:
            source = 'Strategy position checker'
            title = e.message
            send_exception_to_rule_checker(source, title, content, options.rule_checker, ErrorCodes.r0006)
            sys.exit(-1)


if __name__ == '__main__':
    main()
```
