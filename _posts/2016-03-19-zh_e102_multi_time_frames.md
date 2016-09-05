---
title: 多时间周期策略
layout: post
category: root
language: zh
---

本示例展示如何订阅不同周期的K线。你需要在策略的`__init__`方法的`periods`参数中列出所有需要的K线周期。


```python
from ctxalgoctp.ctp.backtesting_utils import *
```

在`periods`中，我们列出了两个周期，15分钟和30分钟。和之前一样，`strategy_period`定义了交易所在的周期（我们在
此示例中不包括交易的代码）。

config对象的内容会被用来初始化策略对象。


```python
config = {
    'instrument_ids': ['IF99'],                      # We are trading this future instrument.
    'periods': [Periodicity.FIFTEEN_MINUTE, Periodicity.THIRTY_MINUTE],
}
```

以下是策略代码。当一根或多根K线到来的时候，`on_bar`方法会被调用，其参数中包括了所生成的所有的K线的信息。


```python
class MultiTimeFrameStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, periods=periods, description=description, logger=logger)

    def on_bar(self, instrument_id, bars, tick):
        # We do not do trading here, instead, we only print out the arrived ohlc bars.
        # Since we only asked for time-based ohlcs, we get them from bars[self.time_based].
        for period, bar in bars[self.time_based].items():
            print('Period={}\tTime={}\tClose price={}'.format(Periodicity.name(period), bar.timestamp, bar.close))
```

现在我们对以上交易策略进行历史数据的回测。


```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.
report, data_source = backtest(MultiTimeFrameStrategy, config, start_date, end_date)

```
