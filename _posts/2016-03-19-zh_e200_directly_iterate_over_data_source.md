---
title: 直接迭代访问回测数据
layout: post
category: zh
---

本示例展示如何直接对回测历史数据进行迭代访问，以及如何设置需要对哪些周期的K线数据进行迭代访问。


```python
import os
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.trading_utils.future_info_calculator_factory import FutureInfoCalculatorFactory
from ctxalgoctp.ctp.backtesting_utils import get_data_source

```

## 迭代访问单一交易品种的数据

首先，我们从CTX服务器获取历史交易数据。下载的数据会被存放在`base_folder`文件夹中。你可以指定其它的文件夹。
需要获取历史数据的品种在`instrument_ids`中指定。现在我们指定一个品种。历史数据的起止时间由`start_date`和`end_date`指定。
历史数据的周期由`data_period`来指定，这个周期应该是以后需要生成的K线周期中最小的。在此，我们指定历史数据的周期为15分钟。

在调用`get_data_source`后，历史数据就可以从`data_source`中获得。


```python
instrument_ids = ['cu99']
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.
base_folder = os.path.join(os.environ['CTXALGO_TEST'], 'strategies', 'iterating_data_source')
data_period = Periodicity.FIFTEEN_MINUTE
data_source = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)
```

接着，我们迭代历史数据。`ohlc_periods`中指定迭代器需要返回哪些周期的K线数据。在这里，我们指定需要返回30分钟和日线级别的
K线。这些级别的K线都是由之前下载的15分钟级别的历史数据生成的。

然后，我们通过调用`bars_iterator`来获得一个K线数据的迭代器。该迭代器返回一个又三个元素组成的元组
(instrumment_id, bars, all_ohlcs)。`instrument_id`指明生成的是哪个品种的K线。`bars`包含着刚生成的K线。`all_ohlcs`包含着
到目前为止生成的所有的K线。


```python
def iterate_over_data(ds, periods):
    """
    Iterate over data_source.
    :param ds: BacktestingDataSource, the data source containing historical trading data.
    :param periods: [Periodicity], the list of periods that the iterator should generate ohlcs for.
    """
    factory = FutureInfoCalculatorFactory()
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

## 迭代访问多个交易品种的数据

以上代码展示的是如何迭代单一品种的交易数据。接下来我们展示如何同时迭代多品种的交易数据。要迭代多品种，唯一需要进行的
代码修改是通过`instrument_ids`指定多个品种的ID。在这里我们指定了两个品种cu99和rb99。


```python
instrument_ids = ['cu99', 'rb99']
data_source2 = get_data_source(instrument_ids, base_folder, start_date, end_date, data_period)

print('============ Iterate over two instruments\' ohlcs ============')
iterate_over_data(data_source2, ohlc_periods)
```

## 迭代多个品种的K线

如果你需要在多个品种的指定K线同时向前走过了一根K线时得到一个事件，你可以使用`multiple_bars_iterator`。
该方法返回一个迭代器。当所有指定品种都走过一根K线后，这个迭代器会生成一个元素。


```python
print('============ Iterate over multiple bars ============')
for ohlcs in data_source2.multiple_bars_iterator(period=Periodicity.THIRTY_MINUTE, instrument_ids=['cu99', 'rb99']):
    ohlc1 = ohlcs['cu99']
    ohlc2 = ohlcs['rb99']
    print('{} {}: {}\t{} {}: {}'.format('cu99', ohlc1.length, ohlc1.dates[-1], 'rb99', ohlc2.length, ohlc2.dates[-1]))

```

## 直接返回完整的ohlc数据

有时候，你希望直接获得完整的ohlc数据，而不是通过一个迭代器一根K线一根K线的获得。这时候，可以使用`ohlcs`方法。该方法
具有和`bars_iterator`方法一样的参数，所不同的是，`ohlcs`返回生成完的完整的ohlc。


```python
print('============ Get whole ohlcs directly ============')
ohlcs = data_source2.ohlcs(ohlc_periods)
print('30 minute cu99 ohlc length: {}'.format(ohlcs['time-based']['cu99'][Periodicity.THIRTY_MINUTE].length))
print('Daily cu99 ohlc length: {}'.format(ohlcs['time-based']['cu99'][Periodicity.DAILY].length))
print('30 minute rb99 ohlc length: {}'.format(ohlcs['time-based']['rb99'][Periodicity.THIRTY_MINUTE].length))
print('Daily rb99 ohlc length: {}'.format(ohlcs['time-based']['rb99'][Periodicity.DAILY].length))


```
