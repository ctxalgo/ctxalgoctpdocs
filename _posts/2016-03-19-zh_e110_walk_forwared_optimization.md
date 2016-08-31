---
title: 分段前向优化
layout: post
category: root
language: zh
---

本示例展示如何对策略的参数进行分段前向优化。要使用优化，你需要指定要优化的参数，目标函数以及优化器。


```python
from ctxalgolib.ta.optimization.parameters import ParameterVector, OrdinalParameter, DiscreteParameter, ContinuousParameter
from ctxalgolib.ta.optimization.optimization_utils import OptimizerUtils
from ctxalgolib.ta.optimization.regularizer import L2Regularizer
from ctxalgolib.ta.online_indicators.moving_averages import MovingAveragesIndicator
from ctxalgolib.loggers.logger_handlers import ConsoleLoggerHandler
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgoctp.ctp.strategy import StrategyUtils


class TrendFollowingStrategy(AbstractStrategy):
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

        # The last generated open signal.
        self.last_signal = 0

        # Setup the moving average calculators.
        self.ma_ind1 = MovingAveragesIndicator(period=self.parameters['fast_ma_period'])
        self.ma_ind2 = MovingAveragesIndicator(period=self.parameters['slow_ma_period'])

    def calculate_trend(self, ohlc):
        """
        Calculate trend from ohlc.
        :param ohlc: OHLC, ohlc object.
        :return: int, 1 means up-trend, -1 means down trend, 0 means no trend.
        """
        result = 0
        ma_fast = self.ma_ind1.calculate(ohlc)
        ma_slow = self.ma_ind2.calculate(ohlc)
        if ma_fast[-2] < ma_slow[-2] and ma_fast[-1] > ma_slow[-1]:
            result = 1
        elif ma_fast[-2] > ma_slow[-2] and ma_fast[-1] < ma_slow[-1]:
            result = -1
        return result

    def on_bar(self, instrument_id, bars, tick):
        self.last_signal = 0
        if self.strategy_ohlc().length > self.parameters['previous_bars']:
            ohlc = self.strategy_ohlc()
            self.last_signal = self.calculate_trend(ohlc)
            if self.last_signal != 0 and not self.has_pending_order(instrument_id=instrument_id):
                if self.in_market_period(tick['timestamp'], instrument_id=instrument_id, delta=timedelta(minutes=30)):
                    self.change_position_to(1 * self.last_signal)
                self.last_signal = 0


def get_strategy_config(base_folder):
    """
    Get configuration for your strategy.
    :param base_folder: string, the folder used to store strategy logs.
    :return: dict{string: object}, strategy configuration.
    """
    logger = StrategyUtils.get_logger(base_folder, has_console=False, has_file=True)
    config = {
        'instrument_ids': ['IF99'],
        'strategy_period': Periodicity.FIFTEEN_MINUTE,
        'base_folder': base_folder,
        'logger': logger,
        'parameters': {
            'previous_bars': 15,
            'fast_ma_period': 5,
            'slow_ma_period': 15,
            'open_position_time_delta': 15,
        }
    }
    return config


def main():
    base_folder = safe_get_base_folder(folder='walk_forward_trend_following')

    start_date = '2014-01-01'
    end_date = '2014-04-30'

    # Optimize two ordinal parameters, corresponding to fast_ma_period and slow_ma_period.
    # Other kinds of parameters are:
    # DiscreteParameter, where you specify a set of possible values for that parameter.
    # ContinuousParameter, where you specify an optional bound for the parameter.
    param_specs = ParameterVector([
        OrdinalParameter('fast_ma_period', range(5, 20), init_value=5),
        OrdinalParameter('slow_ma_period', range(8, 60), init_value=8)
    ])

    # Objective function. An optimizer always minimizes the objective.
    # So to maximize net_profit, we minimize -net_profit.
    # An objective function takes one or more parameters, such as the net_profit used below.
    # Possible parameter names are defined at ObjectiveFunctionParameters.names.
    objectives = [lambda net_profit: -net_profit]

    # Constraints in form of: constraint_expr <= 0.
    # If you don't want any constraints, set cons to an empty list.
    cons = [lambda fast_ma_period, slow_ma_period: fast_ma_period - slow_ma_period + 1 <= 0]

    # Regularizer: L2 regularizer around the latest parameter values.
    # If you don't want to use regularizer, set it to None.
    x0 = param_specs.get_init_value()
    regularizers = [L2Regularizer(x0[0], 5.0), L2Regularizer(x0[1], 10)]

    # Choose an optimizer, here we choose hill_climbing.
    # Other possible values are:
    # OptimizerUtils.simulated_annealing, OptimizerUtils.brute_force and OptimizerUtils.random_guess
    optimizer = OptimizerUtils.hill_climbing(iterations=3, restarts=1)

    # Config for walk forward.
    training_window = 30
    testing_window = 10
    testing_start_day = 30  # Testing starts from the 30'th day.

    # Perform walk-forward optimization.
    walk_forward_optimize(
        TrendFollowingStrategy, get_strategy_config, base_folder, start_date, end_date,
        optimizer, param_specs, objectives, training_window, testing_window,
        backtest_data_period=Periodicity.ONE_MINUTE,
        constraints=cons, regularizers=regularizers,
        testing_start_day=testing_start_day,
        progress_logger=ConsoleLoggerHandler())


if __name__ == '__main__':
    main()

```
