---
title: 从订单执行器日志同步策略的仓位
layout: post
category: live
language: zh
---

本程序从订单执行器的日志文件中同步策略的仓位。


```python
import os
from optparse import OptionParser
from datetime import datetime
from ctxalgoctp.ctp.trade_executors.position_recovery import PositionRecovery
from ctxalgoctp.ctp.constants import Constants as C


def main():
    parser = OptionParser()
    parser.add_option('--name', type='string', dest='name',
                      help='Name of the strategy whose position needs to be synced. In form of product.strategy.')
    parser.add_option('--base-folder', type='string', dest='base_folder',
                      help='The base folder for the strategy whose position needs to be synced.')
    parser.add_option('--trade-executor', type='string', dest='trade_executor_base_folder',
                      help='The base folder of the trade executor strategy.')
    parser.add_option('--trading-day', type='string', dest='trading_day',
                      help='The trading day in form of yyyymmdd of those positions.')
    parser.add_option('--initial-capital', type='float', dest='initial_capital',
                      help='The initial capital of current account.')
    parser.add_option('--security-company', type='string', dest='security_company', default=None,
                      help='Security company name.')
    parser.add_option('--uuid', type='string', dest='uuid',
                      help='UUID of the source strategy whose position needs to be synced. If None, take the last '
                           'execution of the source strategy from the trade executor log file')
    parser.add_option('--overwrite', action='store_true', dest='overwrite', default=False,
                      help='If present, write the synced position into account.txt in source strategy base folder, '
                           'otherwise, write into a new file in the base folder.')
    parser.add_option('--backup', action='store_true', dest='backup', default=False,
                      help='If present, save the synced account file into the account backup folder.')

    options, args= parser.parse_args()
    source_strategy = options.name
    trading_day = datetime.strptime(options.trading_day, '%Y%m%d').date()
    base_folder = options.base_folder

    recovery = PositionRecovery()
    assert options.uuid is not None
    source_account_path = os.path.join(
        base_folder, C.account_backup_folder_name, C.account_file_with_trading_day(trading_day, options.uuid))
    trade_executor_path = os.path.join(options.trade_executor_base_folder, C.output_file(trading_day, text=True))
    account = recovery.recover(source_strategy, source_account_path, trade_executor_path, source_uuid=options.uuid,
                               initial_capital=options.initial_capital, security_company=options.security_company,
                               trade_executor_log_text=True)
    if options.overwrite:
        account_path = os.path.join(base_folder, C.account_file_name)
    else:
        account_path = os.path.join(base_folder, C.synced_account_file_name)
    account.save_to_file(account_path)

    if options.backup:
        backup_path = os.path.join(
            base_folder, C.account_backup_folder_name, C.account_file_with_trading_day(trading_day))
        account.save_to_file(backup_path)


if __name__ == '__main__':
    main()

```
