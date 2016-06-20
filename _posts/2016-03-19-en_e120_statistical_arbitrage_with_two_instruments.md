---
title: Statistical arbitrage with two instruments
layout: post
category: en
---

```python
from ctxalgoctp.ctp.backtesting_utils import *
import math
from statsmodels.tools.tools import add_constant, Bunch
from statsmodels.regression.linear_model import OLS, yule_walker
from statsmodels.tsa.stattools import adfuller
import numpy as np


def ADF(v, crit='5%', max_d=20, reg='nc', autolag='AIC'):
    """ Augmented Dickey Fuller test

    Parameters
    ----------
    v: ndarray matrix
        residuals matrix

    Returns
    -------
    bool: boolean
        true if v pass the test
    """

    testResult = False
    adf = adfuller(v, max_d, reg, autolag)
    if(adf[0] < adf[4][crit]):
        testResult = True
    else:
        pass

    return testResult


class StatisticalArbitrageStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder, description, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder,
            strategy_period=strategy_period, description=description, logger=logger)

        self.on_before_run_actions.append(self.on_before_run)
        self.should_trade = False
        self.params = None

        self.add_bar_arrival_action(self.on_multiple_bars)

    def on_multiple_bars(self, strategy, ohlc_kind, period, name, ohlcs):
        """
        :param strategy: AbstractStrategy, which will be initialized with the current strategy object.
        :param ohlc_kind: string, the ohlc kind, can be 'time-based', 'volatility-based'.
        :param period: Periodicity in case of time-based ohlc, or int/float as volatility threshold
            in case of volatility-based ohlc.
        :param name: string, the bar arrival action name, same as the name parameter specified here.
        :param ohlcs: dict{string: OHLC}, the list of ohlcs, keys are instrument ids, values are the ohlc objects.
        """
        if self.should_trade:
            ohlc1 = self.ohlc('i99')
            ohlc2 = self.ohlc('rb99')

            if ohlc2.closes[-1] - self.params[0] - self.params[1] * ohlc1.closes[-1] > 2 * self.std:
                self.change_positions_to({'rb99': -1, 'i99': 1})
            else:
                self.change_positions_to({'rb99': 1, 'i99': -1})

    def on_before_run(self, strategy):
        ohlc1 = self.yesterday_ohlc('i99')
        ohlc2 = self.yesterday_ohlc('rb99')

        if ohlc1 is not None and ohlc2 is not None:
            common_dates = set(ohlc1.dates) & set(ohlc2.dates)
            y1 = ohlc1.project_by_dates(common_dates)
            y2 = ohlc2.project_by_dates(common_dates)

            log_closes1 = [math.log(x) for x in y1.closes]
            log_closes2 = [math.log(x) for x in y2.closes]
            log_closes1 = add_constant(log_closes1)
            st1_fit = OLS(log_closes2, log_closes1, hasconst=True).fit()
            st1_resid = st1_fit.resid
            self.should_trade = ADF(st1_resid)
            self.params = st1_fit.params
            self.std = math.sqrt(np.var(st1_resid))


start_date = '2014-01-01'
end_date = '2014-12-31'

config = {
    'instrument_ids': ['i99', 'rb99'],  # Specify multiple instrument ids to trade.
    'strategy_period': Periodicity.ONE_MINUTE,
}

backtest(StatisticalArbitrageStrategy, config, start_date, end_date)

```
