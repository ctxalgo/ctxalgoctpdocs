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
from ctxalgolib.ohlc.padding.ohlc_padder import SimplePadder
from ctxalgolib.ohlc.timestamp_generator import CommodityFutureTimestampGenerator
from ctxalgoctp.ctp.plain_backtester import PlainBacktester
from ctxalgoctp.ctp.slippage_models import VolumeBasedSlippageModel


sids = ['cu00', 'ag00']
folder = 'c:\\tmp\\data'

start_time = datetime(2017, 1, 1)
end_time = datetime(2017, 7, 1)
period = Periodicity.ONE_MINUTE

# Get historical ohlcs.
feed = HistoricalLocalDataFeed(
    folder, sids=sids, period=period,
    start_time=start_time, end_time=end_time,
    profits=True, dominants=True, text_file=True, enforce_fetch=True)

# Pad ohlcs with given timestamp generator.
# This timestamp generator will generate all minute bar (because period is ONE_MINUTE)
# timestamps between start_time and end_time.
timestamp_gen = CommodityFutureTimestampGenerator(start_time, end_time, period)
p = SimplePadder()
ohlcs = p.pad_ohlcs(
    feed.ohlcs(sids, start_time=start_time, end_time=end_time, periodicity=period),
    timestamp_gen)

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




```
