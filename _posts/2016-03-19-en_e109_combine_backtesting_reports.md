---
title: Combining backtesting reports
layout: post
category: root
language: en
---

When you have several strategy backtesting results, sometimes you want to see the combined performance of them, as if
these strategies are running in a strategy portfolio. Use `CompositeStrategyReport` can easily achieve this.
`CompositeStrategyReport` accepts a list of individual backtesting reports and combines them together.
`CompositeStrategyReport` has the same interface as the individual backtesting reports.

```python
import os
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.report.abstract_strategy_report import CompositeStrategyReport
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import backtest
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgoctp.ctp.backtesting_utils import safe_get_base_folder

```

Here, we execute two backtests, and then use the two individual backtesting reports to construct a composite report.

```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.

individual_reports = []
for instrument_id in ['IF99', 'cu99']:
    config = {
        'instrument_ids': [instrument_id],
        'periods': [Periodicity.FIFTEEN_MINUTE],
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        },
        'base_folder': safe_get_base_folder(folder='composite_reports' + '_' + instrument_id)
    }

    # Backtest the current strategy, and collect individual reports.
    report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)
    individual_reports.append(report)

# Construct a composite backtesting report from individual ones.
composite_report = CompositeStrategyReport(reports=individual_reports)

```

Now, we can visualize the composite report just as how we visualize individual reports.

```python
c = ChartsWithReport(composite_report, None, open_in_browser=True)
c.range(7)  # Zoom out to view the whole chart.
c.balance(net_value=False, height=500, position='a1')  # Draw the balance.
c.drawdowns(max_drawdown_color='pink', position='a1')  # Visualize the max-drawdown period(s).
c.instrument_positions(instrument_id='IF99')  # Draw the positions for IF99.
c.instrument_positions(instrument_id='cu99')  # Draw the positions for cu99.
c.show()  # Open the html page in browser.



```
