---
title: Basic example on trading relative instrument ids
layout: post
category: live
language: en
---


```python
import os
from termcolor import colored
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgoctp.ctp.live.r100_connect_to_ctp import JustConnect
from ctxalgoctp.ctp.dominant_provider import DominantProvider
from ctxalgoctp.ctp.strategy_utils import StrategyUtils
from ctxalgoctp.ctp.tests.simulation_accounts import CtpSimulationAccounts
from ctxalgoctp.ctp.factories.ctp_real_factory import CtpRealFactory
from ctxalgoctp.ctp.backtesting_utils import safe_get_base_folder


def main():
    # This scripts connects to the given trading account and display all held positions.

    # TODO: You need to change these values according to when you run this script.
    dp = DominantProvider({
        'SR00': 'SR609',
        'cu00': 'cu1609',
        'cu01': 'cu1610',
    })

    file_base_name = os.path.basename(os.path.splitext(__file__)[0])
    base_folder = safe_get_base_folder(folder=file_base_name)
    config = {
        'base_folder': base_folder,
        'strategy_period': Periodicity.ONE_MINUTE,
        'instrument_ids': ['cu00'],
        'parameters': {
            'wait_for_ohlc': False,   # Specify if the strategy terminates after we have received some ohlc bars.
            'ohlc_bar_count': 70      # Specify how many ohlc bars for all instruments that we are waiting for.
        },
        'logger': StrategyUtils.get_logger(base_folder, has_console=True, has_file=True, append=True),
        'use_remote_account': True,
    }

    # Initialize the strategy.
    account = CtpSimulationAccounts.get_account('simnow_future')
    ctp_factory = CtpRealFactory.create(account=account)
    ctp_factory.set_dominant_provider(dp)

    strategy = JustConnect(**config)
    strategy.set_ctp_factory(ctp_factory)
    strategy.run()


if __name__ == '__main__':
    main()



```
