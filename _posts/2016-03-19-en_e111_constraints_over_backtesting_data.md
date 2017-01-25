---
title: Constructing new ohlc data for backtesting
layout: post
category: root
language: en
---

When backtesting a strategy, sometimes you need to modify an existing ohlc object to make certain condition met,
for example, you want to change the close price of an ohlc bar to enforce a stop loss signal. This example shows
how to use constraints to generate a new ohlc object from an existing ohlc object. Thew newly constructed ohlc data
is similar to the original ohlc, but with the constraints met.

To run the example, you need to setup the Z3 solver, following instructions from ctxalgolib.ohlc.constraint_based_ohlc_generator.ConstraintBasedOhlcGenerator
to setup Z3.


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
