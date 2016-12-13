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
import zmq
from time import sleep
import json
import traceback
import sys
from ctxalgolib.rule_checking.configs import RuleCheckingConfigs
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils as Topics
from ctxalgolib.mission_control.mission_control import MissionController
from ctxalgoctp.ctp.trade_executors.position_recovery import PositionRecovery
from ctxalgoctp.ctp.constants import Constants as C
from ctxalgoctp.ctp.strategy_utils import StrategyUtils
from ctxalgoctp.ctp.position_utils import PositionUtils


def communicate(mission_controller, trade_executor, source_with_uuid, info, max_tries=1, wait=5):
    """
    Communicate with the trade executor.
    :param mission_controller: MissionController, the mission controller.
    :param trade_executor: string, name of the trade executor strategy, in form of product.strategy.
    :param source_with_uuid: string, the full signature of the crashed strategy, in form of product.strategy#uuid.
    :param info: string, the message kind.
    :param max_tries: int, the max number of tries to send message to trade executor.
    :param wait: int, the max number of seconds to wait for a message to be replied.
    :return: dict{string: object}, the reply of the message, None if no reply has been received before tiemout.
    """
    tries = 0
    msg = '{}:{}'.format(info, source_with_uuid)
    source = source_with_uuid
    while tries < max_tries:
        print('Send: {}'.format(msg))
        mission_controller.send_to_strategy(trade_executor, str(msg))
        sleep(wait)
        for i in range(10):
            replies = mission_controller.poll_from_strategy(timeout=1)
            if trade_executor in replies:
                for reply in replies[trade_executor]:
                    print('reply: {}'.format(reply))
                    try:
                        data = json.loads(reply)
                        if 'kind' in data and data['kind'] == info and 'source' in data and data['source'] == source:
                            return data
                    except Exception as e:
                        print(traceback.format_exc())
        tries += 1
    return None


def check_trade_executor_status(mission_control, trader_name, source_with_uuid):
    """
    Confirm communication with the trade executor strategy (if it is still running).
    :param mission_control: string, the mission control queue address prefix (without port).
    :param trader_name: string, in form of product.strategy, the name of the trade executor strategy.
    :param source_with_uuid: string, the full signature of the crashed strategy, in form of product.strategy#uuid.
    :return: int, if -1 meaning that there are still pending orders of the crashed strategy, no position sync
        should be carried on. Otherwise, return 0.
    """
    result = 0
    context = zmq.Context()
    c = RuleCheckingConfigs(mission_control)
    mc = MissionController(
        context, queues_from_strategies=c.front_proxy_output_queue(), queues_into_strategies=c.back_proxy_input_queue(),
        proxy=True, strategy_name=trader_name)
    # Sleep for a few seconds to reduce the problem of zmq slow-joiner.
    sleep(5)
    source = source_with_uuid
    reply1 = communicate(mc, trader_name, source, 'info', max_tries=3)
    if reply1 is None:
        print('Cannot connect to the trade executor, assume no pending orders from the crashed strategy.')
    else:
        reply2 = communicate(mc, trader_name, source, 'cancel', max_tries=1)
        if reply2 is not None and reply2['canceled_orders'] != 0:
            sleep(5)
            reply3 = communicate(mc, trader_name, source, 'has_pending_order', max_tries=1)
            if reply3 is None or reply3['pending']:
                print('Crashed strategy still has pending orders.')
                result = -1
            else:
                reply4 = communicate(mc, trader_name, source, 'flush_log', max_tries=1)
    mc.dispose()
    context.term()
    return result


def get_cmd_parser():
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

    parser.add_option('--mission-control', type='string', dest='mission_control', default=None,
                      help='Specify the mission-control to the trade executor. If given, try to talk to the trade '
                           'executor to cancel all pending orders from the crashed strategy. If not given, assume '
                           'there is already no pending orders from the crashed strategy.')

    parser.add_option('--initial-capital', type='float', dest='initial_capital', default=None,
                      help='The initial capital of current account.')
    parser.add_option('--security-company', type='string', dest='security_company', default=None,
                      help='Security company name.')

    parser.add_option('--overwrite', action='store_true', dest='overwrite', default=False,
                      help='If present, write the synced position into account.txt in source strategy base folder, '
                           'otherwise, write into a new file in the base folder.')
    return parser


def main():
    parser = get_cmd_parser()
    options, args = parser.parse_args()

    assert os.path.exists(options.strategy), '{} must exists.'.format(options.strategy)
    assert os.path.exists(options.trade_executor), '{} must exists.'.format(options.trade_executor)

    recovery = PositionRecovery()
    if os.path.isdir(options.strategy):
        base_folder = options.strategy
        uuids = recovery.find_last_execution_uuids(base_folder, 2)
        if len(uuids) == 0:
            print('No execution session of the strategy is found. Do nothing and exit.')
            sys.exit(0)
        elif len(uuids) == 1:
            # Only one execution session is found. This means that the strategy crashes in its first execution.
            # So we need to get the positions before the first execution, which is stored in pre_account.txt,
            # and then apply trades on those positions.
            old_account_path = os.path.join(base_folder, C.old_account_file_name)
            if os.path.exists(old_account_path):
                account_backup_file = old_account_path
                crashed_uuid = uuids[0]
            else:
                print('No old_account.txt file is found. Do nothing and terminate.')
                sys.exit(0)
        else:
            # Found the last two strategy execution sessions. The last session is treated as the crashed session.
            # The second to the last session is treated as the corrected session. Of course, the second to the last
            # session can be a synced session as well.
            correct_uuid = uuids[0]
            account_backup_file = recovery.find_account_backup_file(base_folder, correct_uuid)
            assert account_backup_file is not None, 'Account file with uuid {} must exist.'.format(correct_uuid)
            crashed_uuid = uuids[1]  # The second element is the crashed uuid.
    else:
        account_backup_file = options.strategy
        base_folder = StrategyUtils.base_folder_from_account_backup_file(account_backup_file)
        crashed_uuid = options.strategy_uuid

    if os.path.isdir(options.trade_executor):
        # Given trade_executor is the base folder for the trade executor strategy, heuristically find the log file
        # and name of the trade executor strategy.
        trader_base_folder = options.trade_executor
        trade_executor_log_file, trader_name = recovery.find_trader_strategy_log_with_source_uuid(
            trader_base_folder, crashed_uuid)
    else:
        # Given trade_executor is the log file for the trade executor strategy,
        # only need to find the trade executor strategy name.
        trade_executor_log_file = options.trade_executor
        trader_name = recovery.find_trade_executor_name(trade_executor_log_file, crashed_uuid)

    if trade_executor_log_file is None or trader_name is None:
        print('No trades found from trade executor log file {}. Nothing to do.'.format(trade_executor_log_file))
    else:
        trader_name = str(trader_name)
        source_strategy = recovery.find_strategy_name(base_folder, crashed_uuid)
        source_with_uuid = str(Topics.strategy_with_uuid(source_strategy, crashed_uuid))

        # If mission control is given, try to communicate with the trade executor to see if there are pending orders
        # of the crashed strategy. If the trade executor is still running (it can be connected), and there are pending
        # orders of the crashed strategy, we should not carry out position synchronization.
        if options.mission_control is not None:
            code = check_trade_executor_status(options.mission_control, trader_name, source_with_uuid)
            if code != 0:
                print('There are still pending orders from the crashed strategy {} inside the trade executor {}. '
                      'No position synchronization should be carried out. Please make sure that all the pending orders '
                      'have been cancelled first.'.format(source_with_uuid, trader_name))
                sys.exit(code)

        log_file_name = os.path.split(trade_executor_log_file)[-1]
        crashed_trading_day = C.parse_log_file(log_file_name)
        account, applied_trades = recovery.recover(
            source_strategy, account_backup_file, trade_executor_log_file, crashed_source_uuid=crashed_uuid,
            initial_capital=options.initial_capital, security_company=options.security_company)

        account_file_name = C.account_file_name if options.overwrite else C.synced_account_file_name
        account_path = os.path.join(base_folder, account_file_name)

        print('Applied {} trades, new account file saved at {}'.format(len(applied_trades), account_path))
        if not options.overwrite:
            print('Note: the synced position file is stored as {}, to make it have effect, you need to run current '
                  'script with the --overwrite command line option.'.format(C.synced_account_file_name))

        # Save the sync position in base folder, and save the synced position file in account backup folder,
        # with the crashed uuid. This way, in later crash syncs, this new account file can be used as starting point.
        PositionUtils.save_account(account, crashed_uuid, base_folder, crashed_trading_day, options.overwrite)


if __name__ == '__main__':
    main()



```
