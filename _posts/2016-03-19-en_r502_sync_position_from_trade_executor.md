---
title: Synchronize strategy position from log of trade executor
layout: post
category: live
language: en
---

This script synchronizes a strategy's position from log file of the associated trade executor.

```python
import os
from optparse import OptionParser
from datetime import datetime
from ctxalgoctp.ctp.trade_executors.position_recovery import PositionRecovery
from ctxalgoctp.ctp.constants import Constants as C
from ctxalgoctp.ctp.strategy_utils import StrategyUtils


def main():
    parser = OptionParser()
    parser.add_option('--strategy', type='string', dest='strategy',
                      help='Case 1. Path to the account file of the strategy whose positions are to be sync from its '
                           'associated trade executor. The list of account files are located in the account_backup '
                           'sub-folder inside the strategy base folder. The account file name contains trading day '
                           'and timestamp information, Every time when '
                           'a strategy is executed, it will have as unique uuid. You should specify the last account '
                           'file on which the strategy has terminated successfully. This is because the sync will '
                           'take all the trades that happens after the last strategy successful run from the trade '
                           'executor and apply the trades to the last correct strategy positions, which are from '
                           'the account file on which the strategy succeeded the last time. '
                           'Case 2. If the value is a folder, it is interpreted as the base folder of the strategy. '
                           'The recoverer will look for the strategy log and assume that the last execution session '
                           'is the crashed session, and the next to the last execution session succeeded. And the '
                           'account file associated with the next to the last execution session contains the last '
                           'known positions. So in case 2, the recover will heuristically find the account backup file '
                           'containing the last known current positions and will find the uuid of the crashed '
                           'execution session. In other words, --strategy BASE_FOLDER is equal to '
                           '--strategy ACCOUNT_BACKUP_FILE --strategy-uuid UUID.')

    parser.add_option('--strategy-uuid', type='string', dest='strategy_uuid',
                      help='The uuid of the strategy session which crashed. Every time when a strategy is executed, '
                           'it is given a unique uuid. Here, you specify the uuid of the strategy execution which was '
                           'crashed, hence you need to sync positions from the trade executor. This uuid is used to '
                           'locate trades initialized from the strategy inside the trade executor log file. You can '
                           'find this uuid from the last log file from the strategy execution, which are located in '
                           'the sub-folder log inside the strategy base folder. You should look at the last log file '
                           'because the last log file records the crashing execution session.')

    parser.add_option('--trade-executor', type='string', dest='trade_executor',
                      help='Case 1. Path to the trade executor log file which contains the crashed source '
                           'strategy session. Trades from the source execution session will be extracted and applied '
                           'to the last known source strategy positions to do position recovery. '
                           'Case 2. If the given value is a folder, it is interpreted as the base folder of the '
                           'trade executor. Then the log file containing the crashed source strategy session is '
                           'heuristically detected and used.')

    parser.add_option('--initial-capital', type='float', dest='initial_capital', default=None,
                      help='The initial capital of current account.')
    parser.add_option('--security-company', type='string', dest='security_company', default=None,
                      help='Security company name.')

    parser.add_option('--overwrite', action='store_true', dest='overwrite', default=False,
                      help='If present, write the synced position into account.txt in source strategy base folder, '
                           'otherwise, write into a new file in the base folder.')
    parser.add_option('--backup', action='store_true', dest='backup', default=False,
                      help='If present, save the synced account file into the account backup folder.')

    options, args = parser.parse_args()

    assert os.path.exists(options.strategy), '{} must exists.'.format(options.strategy)
    assert os.path.exists(options.trade_executor), '{} must exists.'.format(options.trade_executor)

    recovery = PositionRecovery()
    if os.path.isdir(options.strategy):
        base_folder = options.strategy
        uuids = recovery.find_last_two_execution_uuids(base_folder)
        assert len(uuids) == 2, 'There must be at least two strategy executions.'
        correct_uuid = uuids[0]
        account_backup_file = recovery.find_account_backup_file(base_folder, correct_uuid)
        assert account_backup_file is not None, 'Account file with uuid {} must exist.'.format(correct_uuid)
        crashed_uuid = uuids[1]  # The second element is the crashed uuid.
    else:
        account_backup_file = options.strategy
        base_folder = StrategyUtils.base_folder_from_account_backup_file(account_backup_file)
        crashed_uuid = options.strategy_uuid

    if os.path.isdir(options.trade_executor):
        trader_base_folder = options.trade_executor
        trade_executor_log_file = recovery.find_trader_strategy_log_with_source_uuid(trader_base_folder, crashed_uuid)
    else:
        trade_executor_log_file = options.trade_executor

    source_strategy = recovery.find_strategy_name(base_folder, crashed_uuid)
    log_file_name = os.path.split(trade_executor_log_file)[-1]
    crashed_trading_day = C.parse_log_file(log_file_name)
    account = recovery.recover(source_strategy, account_backup_file, trade_executor_log_file,
                               crashed_source_uuid=crashed_uuid, initial_capital=options.initial_capital,
                               security_company=options.security_company)

    account_path = os.path.join(base_folder, C.account_file_name if options.overwrite else C.synced_account_file_name)
    account.save_to_file(account_path)

    if options.backup:
        backup_path = os.path.join(
            base_folder, C.account_backup_folder_name,
            C.account_file_with_trading_day(crashed_trading_day, uuid=crashed_uuid, now=datetime.now()))
        account.save_to_file(backup_path)


if __name__ == '__main__':
    main()



```
