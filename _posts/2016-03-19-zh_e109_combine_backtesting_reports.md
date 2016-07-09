---
title: 策略组合回测报告
layout: post
category: root
language: zh
---

当你有多个不同的策略回测报告的时候，你可以把它们当成一个策略组合进行分析。使用`CompositeStrategyReport`达到这一目的。
`CompositeStrategyReport`接受一组单独的回测报告，并构建一个组合报告。你可以像访问单独回测报告一样访问组合报告。


```python
import os
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.report.abstract_strategy_report import CompositeStrategyReport
from ctxalgoctp.ctp.docs.starterkit.e100_trend_following_strategy import TrendFollowingStrategy
from ctxalgoctp.ctp.backtesting_utils import backtest
from ctxalgolib.charting.charts import ChartsWithReport

```

我们在此进行两个单独的策略回测，然后用得到的两个回测报告生成一个组合报告。


```python
start_date = '2014-01-01'  # Backtesting start date.
end_date = '2014-12-31'    # Backtesting end date.

individual_reports = []
for instrument_id in ['IF99', 'cu99']:
    config = {
        'instrument_ids': [instrument_id],
        'strategy_period': Periodicity.FIFTEEN_MINUTE,
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        },
        'base_folder': os.path.join(os.environ['CTXALGO_TEST'], 'strategies', 'composite_reports', instrument_id)
    }

    # Backtest the current strategy, and collect individual reports.
    report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)
    individual_reports.append(report)

# Construct a composite backtesting report from individual ones.
composite_report = CompositeStrategyReport(reports=individual_reports)

```

现在，你可以和之前一样对这个组合回测报告进行绘图分析。


```python
c = ChartsWithReport(composite_report, None, open_in_browser=True)
c.range(7)  # Zoom out to view the whole chart.
c.balance(net_value=False, height=500, position='a1')  # Draw the balance.
c.drawdowns(max_drawdown_color='pink', position='a1')  # Visualize the max-drawdown period(s).
c.instrument_positions(instrument_id='IF99')  # Draw the positions for IF99.
c.instrument_positions(instrument_id='cu99')  # Draw the positions for cu99.
c.show()  # Open the html page in browser.



```
