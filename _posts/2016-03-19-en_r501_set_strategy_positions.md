---
title: Set external positions into strategy.
layout: post
category: live
language: en
---

This script sets the given positions from some external source into a strategy. After such updates, the strategy
will forget its current positions and use only the external positions.

```python
from optparse import OptionParser
from datetime import datetime, time

from ctxalgolib.trading_utils.instrument_utils import InstrumentUtils
from ctxalgolib.trading_utils.future_info_calculator_factory import FutureInfoCalculatorFactory
from ctxalgoctp.ctp.trading_account import TradingAccount
from ctxalgoctp.ctp.trading_account_loaders.local_trading_account_loader import LocalTradingAccountLoader


def parse_position(path):
    iu = InstrumentUtils()
    positions = {}
    with open(path) as f:
        while True:
            line = f.readline()
            if len(line) == 0:
                break
            parts = line.split(',')
            sid = iu.correct_instrument_ids([parts[0].strip()])[0]
            direction = parts[1].strip()
            volume = int(parts[2].strip())
            average_price = float(parts[3].strip())
            if sid not in positions:
                positions[sid] = {}
            positions[sid][direction] = {
                'volume': volume,
                'average_price': average_price
            }
    return positions


def main():
    parser = OptionParser()
    parser.add_option('--base-folder', type='string', dest='base_folder',
                      help='The base folder for the strategy whose position needs to be set.')
    parser.add_option('--position', type='string', dest='position',
                      help='Path to a file containing position information. '
                           'The file is a csv file, each row contains the following comma-separated columns: '
                           'instrument_id, direction, volume, average_price.')
    parser.add_option('--trading-day', type='string', dest='trading_day',
                      help='The trading day in form of yyyymmdd of those positions.')
    parser.add_option('--initial-capital', type='float', dest='initial_capital',
                      help='The initial capital of current account.')
    parser.add_option('--security-company', type='string', dest='security_company', default=None,
                      help='Security company name.')
    options, args= parser.parse_args()

    trading_day = datetime.strptime(options.trading_day, '%Y%m%d').date()
    base_folder = options.base_folder
    security_company = options.security_company

    # Create positions into account.
    future_info_fac = FutureInfoCalculatorFactory(security_company)
    account = TradingAccount(security_company=security_company, available=options.initial_capital)
    trade_id = 0
    for sid, pos in parse_position(options.position).items():
        for direction, pos2 in pos.items():
            price = pos2['average_price']
            volume = pos2['volume']
            trade_id -= 1
            order_ref = 'sync_order_ref_' + str(trade_id)
            timestamp = datetime.combine(trading_day, time(14, 50))
            reason = ''
            volume_multiple = future_info_fac.volume_multiple_calculator().volume_multiple(sid, trading_day)
            margin_rate = future_info_fac.margin_calculator().margin(sid, direction, trading_day)
            account.open_positions(
                sid, price, volume, trade_id, order_ref, timestamp, reason, trading_day, sid,
                volume_multiple=volume_multiple, margin_rate=margin_rate,
                check_available=False)

    # Save account into file.
    loader = LocalTradingAccountLoader(future_info_fac)
    loader.save(account, None, base_folder)


if __name__ == '__main__':
    main()

```
