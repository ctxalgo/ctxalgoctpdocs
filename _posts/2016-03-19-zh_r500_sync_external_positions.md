---
title: 同步外部交易带来的持仓
layout: post
category: live
language: zh
---

本程序负责把外部交易产生的持仓植入到CTX的策略中去。它应用于以下场景：你有一些CTX策略，它们使用同一个交易账户。由于某种
原因，你使用外部的方式进行了一些交易，产生了一些持仓，你想把这些外部的持仓植入到CTX策略中去。这些外部产生的持仓叫做
悬挂支持，因外它们不属于任何CTX策略。

你需要在相关的CTX策略都停止后才能执行该脚本。


```python
from ctxalgolib.output.texttable import Texttable
from ctxalgoctp.ctp.live_strategy_utils import *
from ctxalgoctp.ctp.trading_account_loaders.local_trading_account_loader import LocalTradingAccountLoader


class SynchronizeExternalPositions(AbstractStrategy):
    """
    This strategy's sole purpose is to get the account's positions. Those position information is ready after
    strategy initialization, so we can set the strategy to terminate in the on_before_run method.
    """
    def on_before_run(self, strategy):
        display_positions(self)
        self.set_should_exit(True)


def parse_strategies(options):
    """
    Parse strategies specs from command line options.
    :param options: Command line options.
    :return: dict{string: string}, meaning dict{strategy_name: base_folder}.
    """
    result = OrderedDict()
    for s in options.strategies:
        pos = s.find(':')
        strategy_name = s[:pos]
        base_folder = s[pos+1:]
        assert strategy_name not in result, 'Strategy name cannot be repeated: {}'.format(strategy_name)
        result[strategy_name] = base_folder
    return result


def parse_quota(options, strategies):
    """
    Parse position quota specs.
    :param options: Command line options.
    :param strategies: [string], the list of strategy names.
    :return: dict{string: dict{string: dict{string: int}}}, meaning
        dict{strategy_name: dict{actual_instrument_id: dict{direction: volume}}}.
    """
    result = {}
    for q in options.quota:
        strategy_name = q[:q.find(':')]
        remaining = q[q.find(':')+1:]
        instrument_parts = remaining.split(',')
        pos_dict = {}
        for p in instrument_parts:
            positions = p.split(':')
            instrument_id = positions[0]
            assert instrument_id not in pos_dict, 'instrument cannot be repeated: {}'.format(instrument_id)
            pos_dict[instrument_id] = {'long': None, 'short': None}
            for volume_str in positions[1:]:
                volume = int(volume_str)
                if volume > 0:
                    pos_dict[instrument_id]['long'] = volume
                elif volume < 0:
                    pos_dict[instrument_id]['short'] = volume
                else:
                    assert False, 'Position cannot be zero: {}'.format(instrument_id)
        result[strategy_name] = pos_dict

    for s in strategies:
        if s not in result:
            result[s] = {}
    return result


def parse_initial_capital(options, strategies, default_capital=1000000.0):
    """
    Parse strategy initial capital from command line options.
    :param options: command line options.
    :param strategies: [string], the list of strategy names.
    :param default_capital: float, default initial capital for a strategy.
    :return: dict{string: float}, meaning dict{strategy_name: inital_capital}.
    """
    result = {}
    if options.strategy_initial_capitals is not None:
        for c in options.strategy_initial_capitals:
            parts = c.split(':')
            strategy_name = parts[0]
            capital = float(parts[1])
            assert strategy_name not in result, 'Strategy cannot be repeated: {}'.format(strategy_name)
            result[strategy_name] = capital

    for s in strategies:
        if s not in result:
            result[s] = default_capital

    return result


def load_strategy_accounts(strategies, initial_capitals, ctp_factory):
    """
    Load strategy account position information.
    :param strategies: dict{string: string}, meaning dict{strategy_name: base_folder}.
    :param initial_capitals: dict{string: float}, meaning dict{string_name: initial capital}.
    :param ctp_factory: CtpRealFactory, ctp factory.
    :return: dict{string: TradingAccount}, keys are strategy names. Values are loaded accounts.
        If the account information for a strategy is not available, create a default account, which contains
        no positions, and whose initial capital is from initial_capitals.
    """
    loader = LocalTradingAccountLoader(ctp_factory.future_info_factory())
    result = {}
    for strategy_name, base_folder in strategies.items():
        account = loader.load(ctp_factory, base_folder, initial_capital=initial_capitals[strategy_name])
        result[strategy_name] = account
    return result


def calculate_strategy_positions(accounts, trading_day):
    """
    Calculate the positions from the loaded strategies.
    :param accounts: dict{string: TradingAccount}, meaning dict{strategy_name: TradingAccont}, this is the result
        from load_strategy_accounts.
    :param trading_day: date, current trading day.
    :return: dict{string: dict{string: dict{string: dict{string: dict{string: int}}}}}, meaning
        dict{string_name: dict{instrument_id: dict{actual_instrument_id: dict{direction: dict{historical_or_today: volume}}}}}.
    """
    result = {}
    for s, account in accounts.items():
        result[s] = {}
        for sid, actual_pos in account.positions.items():
            if sid not in result[s]:
                result[s][sid] = {}
            for actual_sid, direction_pos in actual_pos.items():
                if actual_sid not in result[s][sid]:
                    result[s][sid][actual_sid] = {}
                for direction, pos in direction_pos.items():
                    if direction not in result[s][sid][actual_sid]:
                        result[s][sid][actual_sid][direction] = {
                            'historical': 0,
                            'today': 0
                        }
                    for p_item in pos:
                        if p_item[1]['trading_day'] < trading_day:
                            result[s][sid][actual_sid][direction]['historical'] += p_item[1]['volume']
                        else:
                            result[s][sid][actual_sid][direction]['today'] += p_item[1]['volume']
    return result


def calculate_dangling_positions(strategy_positions, account_positions):
    """
    Return the dangling positions from the account. Also need position information from strategies to
    calculate dangling positions.
    :param strategy_positions: Return value from calculate_strategy_positions.
    :param account_positions: This is the account position information from the trading account.
    :return: dict{string: dict{string: dict{string: int}}},
        meaning dict{actual_instrument_id: dict{direction: dict{historical_or_today: volume}}}.
    """
    # Type dict{string: dict{string: dict{string: int}}}.
    # Meaning dict{actual_sid: {direction: {historical/today: volume}}}
    # This is the position information from strategies.
    strategy_info = {
    }
    for s, positions in strategy_positions.items():
        for sid, actual_pos in positions.items():
            for actual_sid, direction_pos in actual_pos.items():
                if actual_sid not in strategy_info:
                    strategy_info[actual_sid] = {
                        'long': {'historical': 0, 'today': 0},
                        'short': {'historical': 0, 'today': 0},
                    }
                for direction, pos in direction_pos.items():
                    strategy_info[actual_sid][direction]['historical'] += pos['historical']
                    strategy_info[actual_sid][direction]['today'] += pos['today']

    # Type dict{string: dict{string: dict{string: int}}}.
    # Meaning dict{actual_sid: {direction: {historical/today: volume}}}
    # This is the position information from the trading account.
    account_info = {}
    for actual_sid, position_info in account_positions.items():
        account_info[actual_sid] = {
            'long': {'historical': 0, 'today': 0},
            'short': {'historical': 0, 'today': 0},
        }
        for direction in ['long', 'short']:
            if direction in position_info:
                for position in position_info[direction]:
                    historical = position['historical']
                    volume = position['position']
                    if volume != 0:
                        if historical:
                            account_info[actual_sid][direction]['historical'] += volume
                        else:
                            account_info[actual_sid][direction]['today'] += volume

    # Type dict{string: dict{string: dict{string: int}}}.
    # Meaning dict{actual_sid: {direction: {historical/today: volume}}}
    # This is the difference between strategy_info and account_info, or the dangling position information.
    dangling = {}
    for actual_sid, direction_pos in account_info.items():
        dangling[actual_sid] = {
            'long': {'historical': 0, 'today': 0},
            'short': {'historical': 0, 'today': 0},
        }
        for direction, pos in direction_pos.items():
            if actual_sid in strategy_info and direction in strategy_info[actual_sid]:
                dangling[actual_sid][direction]['historical'] = pos['historical'] - strategy_info[actual_sid][direction]['historical']
                dangling[actual_sid][direction]['today'] = pos['today'] - strategy_info[actual_sid][direction]['today']
            else:
                dangling[actual_sid][direction]['historical'] = pos['historical']
                dangling[actual_sid][direction]['today'] = pos['today']

    return dangling


def update_accounts(strategies, accounts, quota, dangling_positions, strategy):
    """
    Trade dangling position information into the strategies.
    :param strategies: dict{string: string}, meaning dict{strategy_name: base_folder}.
    :param accounts: dict{string: TradingAccount}, keys are strategy names.
    :param quota: dict{string: dict{string: dict{string: int}}}, meaning
        dict{strategy_name: dict{actual_instrument_id: dict{direction: volume}}}.
    :param dangling_positions: dict{string: dict{string: dict{string: int}}},
        meaning dict{actual_instrument_id: dict{direction: dict{historical_or_today: volume}}}.
    :param strategy: AbstractStrategy, the trading strategy.
    """
    dangling = dangling_positions
    loader = LocalTradingAccountLoader(strategy.ctp_factory.future_info_factory())
    prices = loader.position_price(strategy.info)
    trade_ids = {}
    for strategy_name, quota_pos in quota.items():
        account = accounts[strategy_name]
        trade_ids[strategy_name] = -1
        for actual_sid, poses in quota_pos.items():
            for direction, volume in poses.items():
                if volume is not None:
                    sign = 1 if direction == 'long' else -1
                    volume_to_trade_abs = abs(volume)
                    dangling_historical_abs = abs(dangling[actual_sid][direction]['historical'])
                    dangling_today_abs = abs(dangling[actual_sid][direction]['today'])
                    if dangling_historical_abs > 0:
                        trade_pos = min(dangling_historical_abs, volume_to_trade_abs)
                        dangling[actual_sid][direction]['historical'] -= sign * trade_pos
                        volume_to_trade_abs -= trade_pos
                        price = prices[actual_sid][direction]['historical']
                        loader.open_positions(account, actual_sid, trade_pos * sign, price, True, strategy, trade_ids[strategy_name])
                        trade_ids[strategy_name] -= 1
                    if volume_to_trade_abs > 0:
                        assert dangling_today_abs >= volume_to_trade_abs
                        trade_pos = min(dangling_today_abs, volume_to_trade_abs)
                        dangling[actual_sid][direction]['today'] -= sign * trade_pos
                        volume_to_trade_abs -= trade_pos
                        price = prices[actual_sid][direction]['today']
                        loader.open_positions(account, actual_sid, trade_pos * sign, price, False, strategy, trade_ids[strategy_name])
                        trade_ids[strategy_name] -= 1

    for strategy_name, account in accounts.items():
        loader.save(account, strategy.ctp_factory, strategies[strategy_name], trading_day=strategy.trading_day())


def setup_command_line_parser(cmd_options):
    parser = get_command_line_parser(strategy_class=SynchronizeExternalPositions, cmd_options=cmd_options)
    parser.add_option(
        '--strategy', type='string', action='append', dest='strategies',
        help=("Specify strategies whose positions are to be synchronized with the trading account. "
              "Format is --strategy name:base_folder. Where name is a strategy name, and base folder is "
              "the folder in which the strategy stores information. If you need to specify multiple strategies, "
              "use multiple --strategy options. Example: --strategy s1:/strategies/s1 --strategy s2:/strategies/s2. "
              "This means that two strategies are specified, s1 and s2. Note that you can name the strategies with "
              "anything, the names are used as references in other command line options, --quota and "
              "--strategy-initial-capital."))
    parser.add_option(
        '--quota', type='string', action='append', dest='quota',
        help=("Specify how many positions from the trading account should be traded into the strategies. "
              "Format --quota [name:instrument:long_position,short_position]+. For example,"
              "--quota s1:IF1609:1:-2,cu1610:-3 --quota s2:SR611:5 means that 1 long dangling IF1609 position "
              "should be traded into strategy s1, and -2 short dangling IF1609 position should be traded into "
              "s1, and -3 short position of cu1610 should be traded into s1. And 5 long position of SR611 should "
              "be traded into the strategy s2. Dangling position means that those positions from the trading account "
              "do not belong to any of the positions."))
    parser.add_option(
        '--strategy-initial-capital', type='string', action='append', dest='strategy_initial_capitals',
        help=("Specify the initial capital for strategies. Format --strategy-initial-capital name:capital."
              "For example --strategy-initial-capital s1:2000000. To specify initial capital for multiple "
              "strategies, use multiple --strategy-initial-capital."))
    parser.add_option(
        '--view-only', action='store_true', dest='view_only', default=False,
        help='If True, only view the positions differences, do not perform synchronization.')
    return parser


def get_table(positions, max_width=120):
    table = Texttable(max_width=max_width)
    table.set_deco(Texttable.HEADER)
    columns = ['SID', 'Direction', 'Yesterday', 'Today', 'All']
    table.set_cols_align(['r'] * len(columns))
    rows = [columns]

    for actual_sid, direction_pos in positions.items():
        for direction, pos in direction_pos.items():
            if pos['historical'] + pos['today'] != 0:
                rows.append([actual_sid, direction, pos['historical'], pos['today'], pos['historical'] + pos['today']])
    table.add_rows(rows)
    return table.draw()


def print_strategy_positions(strategies, strategy_positions, max_width=120):
    for strategy_name in sorted(strategy_positions.keys()):
        positions = strategy_positions[strategy_name]
        print('Strategy {}: {}'.format(strategy_name, strategies[strategy_name]))
        pos2 = {}
        for sid, p1 in positions.items():
            for actual_sid, p2 in p1.items():
                if actual_sid not in pos2:
                    pos2[actual_sid] = {
                        'long': {'historical': 0, 'today': 0},
                        'short': {'historical': 0, 'today': 0},
                    }
                for direction, direction_pos in p2.items():
                    pos2[actual_sid][direction]['historical'] += direction_pos['historical']
                    pos2[actual_sid][direction]['today'] += direction_pos['today']
        if len(pos2) > 0:
            print(get_table(pos2, max_width=max_width))
        else:
            print('No position.')
        print('\n')


def print_dangling_positions(dangling_positions, max_width=120):
    print('Dangling positions:')
    print(get_table(dangling_positions, max_width=max_width))
    print('\n')


def main():
    # The following cmd_options is used for testing purpose, it will be used only when
    # there is no command line options provided in sys.argv.
    cmd_options = '--account simnow_future4 --name test.syn_position ' \
                  '--strategy s1:c:\\tmp\\s1 --strategy s2:c:\\tmp\\s2 ' \
                  '--quota s1:cu1610:2,SR611:-2 --quota s2:cu1610:1,SR611:-3'
    parser = setup_command_line_parser(cmd_options)
    options = parser.parse()
    assert options.strategies is not None, 'Strategies must be specified.'

    # Execute the strategy, whose only purpose is to retrieve the positions from the trading account.
    config = {
        'base_folder': options.get_base_folder(),
        'instrument_ids': options.get_instruments(),
        'parameters': {},
        'description': options.get_description(),
        'logger': options.get_logger()
        }
    strategy = run_live(SynchronizeExternalPositions, config, options)

    strategies = parse_strategies(options)
    initial_capitals = parse_initial_capital(options, strategies.keys())
    accounts = load_strategy_accounts(strategies, initial_capitals, strategy.ctp_factory)
    strategy_positions = calculate_strategy_positions(accounts, strategy.trading_day())
    dangling_positions = calculate_dangling_positions(strategy_positions, strategy.info['account_position_details'])

    print_strategy_positions(strategies, strategy_positions)
    print_dangling_positions(dangling_positions)

    # Perform position update into strategies.
    if not options.view_only:
        quota = parse_quota(options, strategies.keys())
        has_dangling = False
        for actual_sid, pos in dangling_positions.items():
            for direction, direction_pos in pos.items():
                if direction_pos['today'] !=0 or direction_pos['historical'] != 0:
                    has_dangling = True
                    break
        if has_dangling:
            update_accounts(strategies, accounts, quota, dangling_positions, strategy)
            print('Dangling positions are traded into the specified strategies.')
        else:
            print('No dangling positions, nothing to do.')


if __name__ == '__main__':
    main()

```
