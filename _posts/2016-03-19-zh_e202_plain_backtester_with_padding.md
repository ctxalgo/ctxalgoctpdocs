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
from ctxalgolib.portfolio.centroid_portfolio import CentroidPortfolio
from ctxalgolib.portfolio.positionator import Positionator
from ctxalgolib.trading_utils.future_info_calculator import FutureVolumeMultipleCalculator, FutureTickSizeCalculator

from ctxalgoctp.ctp.plain_backtester import PlainBacktester
from ctxalgoctp.ctp.slippage_models import VolumeBasedSlippageModel
from ctxalgoctp.ctp.data_preparation.future_data_preparation import prepare_future_historical_data
from ctxalgolib.data_feed.web_mongo_data_feed import WebMongoDataFeed
from eqbstrategies.future.strategy_base import correct_instrument_ids


def main():
    sids = ['TA00']

    assets = ['au', 'cu', 'v', 'm', 'ta', 'rb', 'p', 'zn', 'al', 'l', 'ru', 'y', 'c', 'cf', 'oi', 'j', 'ag', 'fg', 'rm',
              'jm', 'i', 'jd', 'pp', 'hc', 'cs', 'ma', 'sn', 'ni', 'zc', 'sr', 'bu']
    sids = correct_instrument_ids(map(lambda asset: asset + '00', assets))

    original_data_folder = 'c:\\tmp\\bin_data'
    padded_data_folder = 'c:\\tmp\\bin_padded_data'
    start_time = datetime(2010, 1, 1)
    end_time = datetime(2017, 7, 1)
    period = Periodicity.ONE_MINUTE

    # Uncomment the following code if you need to do data download and padding again.
    # For example, when you want to change the start and end time, period, instruments.
    timestamp_gen = CommodityFutureTimestampGenerator(start_time, end_time, period)
    web_feed = WebMongoDataFeed(base_url='http://eqb.dscloud.biz:5008/')
    prepare_future_historical_data(
        sids, period, start_time, end_time, original_data_folder,
        profits=True, dominants=False,
        text_file=False, enforce_fetch=True,
        timestamp_generator=timestamp_gen, padded_data_folder=padded_data_folder, data_feed=web_feed)

    feed = HistoricalLocalDataFeed(
        padded_data_folder, sids=sids, period=period,
        start_time=start_time, end_time=end_time,
        profits=True, dominants=False, text_file=False, enforce_fetch=False)

    sids = ['TA00']
    ohlcs = feed.ohlcs(sids, start_time=start_time, end_time=end_time, periodicity=period)

    # Find the total number of bars.
    first_sid = ohlcs.keys()[0]
    bars = ohlcs[first_sid].length

    # Setup backtester.
    backtester = PlainBacktester(
        initial_capital=1000000.0 * 10,             # Initial capital
        has_commission=True,                        # Include commission in trading
        slippage_model=VolumeBasedSlippageModel())  # Setup slippage model
    backtester.set_instrument_ids(sids)

    # Use centroid as portfolio builder. You can choose different portfolio builders, look inside ctxalgolib.portfolio.
    port_gen = CentroidPortfolio(sids)

    # A positionator takes a portfolio from a portfolio builder and turns it into positions to trade.
    # Here you specify desired exposure, future volume multiples.
    positionator = Positionator(
        instrument_ids=sids, desired_exposure=1000000.0, volume_multiple_calculator=FutureVolumeMultipleCalculator())

    tick_size = FutureTickSizeCalculator()

    # Iterate over all bars.
    for bar in range(bars):
        ts = ohlcs[first_sid].dates[bar]
        prices = [ohlcs[sid].closes[bar] for sid in sids]
        volumes = [ohlcs[sid].volumes[bar] for sid in sids]
        profits = [ohlcs[sid].profits[bar] for sid in sids]
        # Position change will cost 1 tick size.
        cost = [0.0 if prices[i] == 0 else 1.0 * tick_size.tick_size(sid, ts) / prices[i] for i, sid in enumerate(sids)]

        # Update the plain backtester with latest known price, volume.
        # This is necessary for the plain backtester to maintain information such pnl.
        backtester.update_prices(prices=prices, volumes=volumes, timestamp=ts)

        # Update the profit series of the portfolio.
        port_gen.update_profits(profits, ts)

        # Dummy strategy: change position at 11:00 every day.
        if ts.time() == time(11, 0):
            portfolio = port_gen.update([1, 0], cost=cost)
            positions = [int(p) for p in positionator.positions(portfolio, prices, ts)]
            backtester.change_positions_to(prices=prices, new_positions=positions, timestamp=ts)

    # Print the backtest report.
    print(backtester.report())


if __name__ == "__main__":
    main()
    # Use the following line if you want to profile your code.
    # profile(main)





```
