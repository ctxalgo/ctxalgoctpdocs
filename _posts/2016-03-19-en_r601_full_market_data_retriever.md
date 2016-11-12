---
title: Retrieving full market tick data
layout: post
category: live
language: en
---

This script retrieves tick data from all traded instruments on a trading day and store them in a file.


```python
from ctxalgoctp.ctp.live_strategy_utils import *
from ctxalgolib.data_feed.web_mongo_data_feed import WebMongoDataFeed


class FullMarketTickRetriever(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder, periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)
        self.set_should_terminate_after_market_close(False)
        self.file = None

    def on_before_run(self, strategy):
        # Strategy will terminates at 16:00:00 of the trading day.
        self.set_exit_time(datetime.combine(self.trading_day(), time(16, 0, 0)))
        self.file = open(self.parameters.output, 'w')

    def on_after_run(self, strategy):
        self.file.close()

    def on_tick(self, instrument_id, tick):
        # Reduce the data to store by removing order book level 2~5.
        for i in range(2, 6):
            bid = 'bid_price' + str(i)
            ask = 'ask_price' + str(i)
            if bid in tick:
                del tick[bid]
            if ask in tick:
                del tick[ask]
        self.file.write(str(self.now()) + '\t' + instrument_id + '\t' + str(tick) + '\n')


def main():
    now = datetime.now()
    # check_should_trade makes sure that code only gets executed on trading days.

    fu = FutureTradingUtils()
    trading_day = fu.trading_day_from_now(now)
    parser = get_command_line_parser(strategy_class=FullMarketTickRetriever, cmd_options=None)
    parser.add_option(
        '--folder', type='string', dest='folder', default=None, help='Folder to store tick data.')
    parser.add_option(
        '--extract', type='string', dest='extract', default=None, help='Comma separated instrument ids to extract.')
    parser.add_option(
        '--trading-day', type='string', dest='trading_day', default=None,
        help='The trading day in form of yyyymmdd to extract data from, only have effect when --extract is given.')

    options = parser.parse()

    if not os.path.exists(options.folder):
        os.makedirs(options.folder)

    if options.extract is None:
        # Run strategy to collect ticks and save in options.folder.
        if check_should_trade(now=now, handle_only_day_trading_days=True):
            config = {
                'base_folder': options.get_base_folder(),
                'instrument_ids': WebMongoDataFeed().instruments(datetime.combine(trading_day, time(0, 0, 0))),
                'parameters': {
                    'output': os.path.join(options.folder, trading_day.strftime('%Y%m%d') + '.txt')
                },
                'description': options.get_description(),
                'logger': options.get_logger()
            }

            run_live(FullMarketTickRetriever, config, options)
    else:
        # Extract ticks.
        instruments = set(options.extract.split(','))
        files = {}
        # Open a file for each instrument to extract, assuming we are not extracting data for too many instruments.
        for sid in instruments:
            path = os.path.join(options.folder, 'extracted_' + sid + '_' + options.trading_day + '.txt')
            files[sid] = open(path, 'w')

        with open(os.path.join(options.folder, options.trading_day + '.txt'), 'r') as f:
            while True:
                line = f.readline().strip()
                if len(line) > 0:
                    parts = line.split('\t')
                    sid = parts[1]
                    if sid in files:
                        files[sid].write(line + '\n')
                else:
                    break

        for f in files.values():
            f.close()


if __name__ == '__main__':
    main()

```
