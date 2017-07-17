---
title: 简化回测器和K线补齐一起使用
layout: post
category: root
language: zh
---



```python
from datetime import datetime, time
from ctxalgolib.data_feed.historical_data_fetcher import HistoricalLocalDataFeed
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.ohlc.timestamp_generator import CommodityFutureTimestampGenerator
from ctxalgolib.profiler_utils import profile
from ctxalgoctp.ctp.plain_backtester import PlainBacktester
from ctxalgoctp.ctp.slippage_models import VolumeBasedSlippageModel
from ctxalgoctp.ctp.backtesting_utils import prepare_historical_data


def main():
    sids = ['cu00', 'ag00']
    sids = ['cu00']
    original_data_folder = 'c:\\tmp\\bin_data'
    padded_data_folder = 'c:\\tmp\\bin_padded_data'
    start_time = datetime(2016, 1, 1)
    end_time = datetime(2017, 7, 1)
    period = Periodicity.ONE_MINUTE

    # Uncomment the following code if you need to do data download and padding again.
    # For example, when you want to change the start and end time, period, instruments.
    # timestamp_gen = CommodityFutureTimestampGenerator(start_time, end_time, period)
    prepare_historical_data(
        sids, period, start_time, end_time, original_data_folder,
        profits=True, dominants=True,
        text_file=False, enforce_fetch=True,
        timestamp_generator=timestamp_gen, padded_data_folder=padded_data_folder)

    # Get historical ohlcs.
    feed = HistoricalLocalDataFeed(
        padded_data_folder, sids=sids, period=period,
        start_time=start_time, end_time=end_time,
        profits=True, dominants=True, text_file=False, enforce_fetch=False)

    ohlcs = feed.ohlcs(sids, start_time=start_time, end_time=end_time, periodicity=period)

    # Find the total number of bars.
    first_sid = ohlcs.keys()[0]
    bars = ohlcs[first_sid].length

    # Setup backtester.
    backtester = PlainBacktester(
        initial_capital=1000000.0,                  # Initial capital
        has_commission=True,                        # Include commission in trading
        slippage_model=VolumeBasedSlippageModel())  # Setup slippage model
    backtester.set_instrument_ids(sids)

    # Iterate over all bars.
    for bar in range(bars):
        ts = ohlcs[first_sid].dates[bar]
        backtester.update_prices(
            prices={sid: ohlcs[sid].closes[bar] for sid in sids},
            volumes={sid: ohlcs[sid].volumes[bar] for sid in sids},
            timestamp=ts)

        # Decide on positions purely based on time, just for illustration.
        position = None
        if ts.time() == time(11, 0):
            position = 1
        elif ts.time() == time(14, 30):
            position = 0

        if position is not None:
            backtester.change_positions_to(
                prices={sid: ohlcs[sid].closes[bar] for sid in sids},
                new_positions={sid: position for sid in sids},
                timestamp=ts)

    # Print the backtest report.
    print(backtester.report())


if __name__ == "__main__":
    main()





```
