---
title: 通过约束条件来构建用于回测的ohlc数据
layout: post
category: root
language: zh
---

在回测一个策略的时候，你有时需要改变原有ohlc的某些值，来使得策略的某些信号被触发。比如你想改变某一根K线的收盘价来使得
某个止损信号被触发。本示例展示如何通过指定约束条件，来从一个原有的ohlc对象生成一个新的ohlc对象。这个新的ohlc对象和原有
的对象很相似，但是指定的约束条件已经被满足了。

要使用这个功能，你需要安装Z3，参见ctxalgolib.ohlc.constraint_based_ohlc_generator.ConstraintBasedOhlcGenerator中的文档
来安装、配置Z3。


```python
from ctxalgolib.ohlc.abstract_ohlc import OhlcAttributes
from ctxalgolib.ohlc.constraint_based_ohlc_generator import OhlcBarConstraint
from ctxalgoctp.ctp.backtesting_utils import *


class DummyStrategy(AbstractStrategy):
    def on_bar(self, instrument_id, bars, tick):
        ohlc = self.ohlc()
        # Check if the specified constraints are met.
        if ohlc.dates[-1] <= datetime(2014, 1, 10, 15, 0):
            assert ohlc.highs[-1] > 3000
            assert ohlc.volumes[-1] == 500


start_date = '2014-01-01'
end_date = '2014-01-31'

# The values in config will be used to instantiate the strategy objects by the backtest method.
config = {
    'instrument_ids': ['cu00'],
    'periods': [Periodicity.FIFTEEN_MINUTE],
}

# Specify constraints to the ohlc data. Here we specify that high values of all bars before 2014-01-10 15:00:00 must
# be larger than 3000. And the volumes of those bars must equal to 500.
# Check ctxalgolib.ohlc.constraint_based_ohlc_generator.ConstraintBasedOhlcGeneratorConstants to see the supported
# ohlc fields, such as highs, volumes, and the supported operators, such as >, =.
# start_date and end_date specifies the range of rows where a constraint should be applied.
# If start_date is None, it means starting from the first ohlc bar. If end_date is None, it means ends at the last bar.
constraints = [
    OhlcBarConstraint(OhlcAttributes.highs, '>', 3000, start_date=None, end_date=datetime(2014, 1, 10, 15, 0)),
    OhlcBarConstraint(OhlcAttributes.volumes, '=', 500, start_date=None, end_date=datetime(2014, 1, 10, 15, 0)),
]

# Backtest with the constraints.
report, data_source = backtest(DummyStrategy, config, start_date, end_date, ohlc_bar_constraints=constraints)

```
