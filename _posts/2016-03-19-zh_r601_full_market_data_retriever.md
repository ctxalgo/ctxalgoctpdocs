---
title: 接收全市场tick数据
layout: post
category: live
language: zh
---

本程序接收全市场的tick数据并将其存到文件中。


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
        self.file.write(str(self.now()) + '\t' + instrument_id + '\t' + str(tick) + '\n')


def main():
    now = datetime.now()
    # check_should_trade makes sure that code only gets executed on trading days.
    if check_should_trade(now=now, handle_only_day_trading_days=True):
        fu = FutureTradingUtils()
        trading_day = fu.trading_day_from_now(now)
        parser = get_command_line_parser(strategy_class=FullMarketTickRetriever, cmd_options=None)
        parser.add_option('--output', type='string', dest='output', default=None, help='Folder to store tick data.')
        options = parser.parse()
        config = {
            'base_folder': options.get_base_folder(),
            'instrument_ids': WebMongoDataFeed().instruments(datetime.combine(trading_day, time(0, 0, 0))),
            'parameters': {
                'output': os.path.join(options.output, trading_day.strftime('%Y%m%d') + '.txt')
            },
            'description': options.get_description(),
            'logger': options.get_logger()
        }

        run_live(FullMarketTickRetriever, config, options)


if __name__ == '__main__':
    main()

```
