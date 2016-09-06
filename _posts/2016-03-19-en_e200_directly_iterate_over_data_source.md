---
title: Iterating over a data source directly
layout: post
category: root
language: en
---

This example shows how to iterate over data source for backtesting directly, without going through the backtester.
But similar to going over data in the backtester, you can setup the iterator to generate different periods of ohlcs.

```python
import os
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.trading_utils.future_info_calculator_factory import FutureInfoCalculatorFactory
from ctxalgoctp.ctp.backtesting_utils import get_data_source, safe_get_base_folder

```

## Iterating over a single instrument's ohlc.

First, we get some historical trading data. The data is downloaded from CTX servers and stored in `base_folder`.
You can get the `base_folder` to your own folder. Use `instrument_ids` to specify the instruments whose data
is to be downloaded. Here, we only specify a single instrument. The historical data time range is decided by
`start_date` and `end_date`. `data_period` specifies the granularity period of the data. It should be smallest
among the periods of the ohlcs that you want your iterator to generate. Here we specify
that we want to download 15 minute bars as historical data.

The historical trading data will be accessible through `data_source`.

```python
instrument_ids = ['cu00']
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.
base_folder = safe_get_base_folder(folder='iterating_data_source')
data_period = Periodicity.FIFTEEN_MINUTE
data_source = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)
```

Next, we iterate over the historical data. `ohlc_periods` specifies the periods of the ohlcs that we want
to iterate over. Here we specify that we want to iterate over two periods of ohlcs, 30 minute bars and daily bars.
These 30 minute and daily bars are generated from the downloaded 15 minute bars.

Then, we call `bars_iterator` to get an iterator which yields the required 30 minute and daily bars. The iterator yields
a tuple of three elements (instrument_id, bars, all_ohlcs). `instrument_id` tells you whose bars are being returned
this time. `bars' contains the returned bars. `all_ohlcs` contains all the ohlcs that are generated so far.

```python
def iterate_over_data(ds, periods):
    """
    Iterate over data_source.
    :param ds: BacktestingDataSource, the data source containing historical trading data.
    :param periods: [Periodicity], the list of periods that the iterator should generate ohlcs for.
    """
    for instrument_id, bars, all_ohlcs in ds.bars_iterator(periods):
        for period, bar in bars['time-based'].items():
            ohlc = all_ohlcs['time-based'][instrument_id][period]
            print('instrument={}\ttimestamp={}\tperiod={}\tohlc_bar_count={}'.format(
                instrument_id, Periodicity.name(period), bar.timestamp, ohlc.length))

    print('')

print('============ Iterate over single instrument\'s ohlcs ============')
ohlc_periods = [Periodicity.THIRTY_MINUTE, Periodicity.DAILY]
iterate_over_data(data_source, ohlc_periods)
```

## Iterating over multiple instruments' ohlcs

The above code demonstrates how to iterate over a single instrument's ohlcs. The following code shows how to
iterate over multiple instruments' ohlcs at the same time. This is useful when designing strategies involving
multiple instruments.

To iterate over multiple instruments, the only change we need is to get data for those instruments. So we
set `instrument_ids` to have two instruments. Here we specify two instruments, cu99 and rb99. That's it.

```python
instrument_ids = ['cu00', 'rb00']
data_source2 = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)

print('============ Iterate over two instruments\' ohlcs ============')
iterate_over_data(data_source2, ohlc_periods)
```

## Iterating over multiple bars

If you want to get an event when a set of instruments have all proceeded with a bar,
you can use `multiple_bars_iterator`. This method returns an iterator. The iterator yields an element
when all the listed instruments progressed with a bar in the given periods.


```python
print('============ Iterate over multiple bars ============')
for ohlcs in data_source2.multiple_bars_iterator(period=Periodicity.THIRTY_MINUTE, instrument_ids=['cu00', 'rb00']):
    ohlc1 = ohlcs['cu00']
    ohlc2 = ohlcs['rb00']
    print('{} {}: {}\t{} {}: {}'.format('cu00', ohlc1.length, ohlc1.dates[-1], 'rb00', ohlc2.length, ohlc2.dates[-1]))

```

## Getting whole ohlcs directly

If you want to get the whole ohlcs directly, instead of iterating over an iterator and get the ohlc gradually,
call the `ohlcs` method from a data source. The `ohlcs` method has the same parameters as the `bars_iterator` method,
but it returns the full ohlcs, instead of an iterator.

```python
print('============ Get whole ohlcs directly ============')
ohlcs = data_source2.ohlcs(ohlc_periods)
print('30 minute cu00 ohlc length: {}'.format(ohlcs['time-based']['cu00'][Periodicity.THIRTY_MINUTE].length))
print('Daily cu99 ohlc length: {}'.format(ohlcs['time-based']['cu00'][Periodicity.DAILY].length))
print('30 minute rb00 ohlc length: {}'.format(ohlcs['time-based']['rb00'][Periodicity.THIRTY_MINUTE].length))
print('Daily rb99 ohlc length: {}'.format(ohlcs['time-based']['rb00'][Periodicity.DAILY].length))


```
