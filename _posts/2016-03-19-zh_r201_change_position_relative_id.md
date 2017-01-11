---
title: 使用相对的品种代码进行调仓
layout: post
category: live
language: zh
---



```python
import os
from termcolor import colored
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgoctp.ctp.live.r101_change_positions import ChangePosition
from ctxalgoctp.ctp.dominant_providers.dominant_provider import DominantProvider
from ctxalgoctp.ctp.dominant_providers.dummy_dominant_provider import DummyDominantProvider
from ctxalgoctp.ctp.strategy_utils import StrategyUtils
from ctxalgoctp.ctp.tests.simulation_accounts import CtpSimulationAccounts
from ctxalgoctp.ctp.factories.ctp_real_factory import CtpRealFactory
from ctxalgoctp.ctp.backtesting_utils import safe_get_base_folder


def main():
    # This scripts demonstrate how to trade with relative instrument ids.
    # Relative instrument id means to use cu00 to mean the dominant instrument of cu, etc.
    # To do that, we set the dominant provider, which defines the mapping from dominant instrument id
    # to actual instrument id. We specify the dominant provider here for ease of demonstration.
    # In an actual trading setting, the dominant provider is specified using our own database.

    # TODO: You need to change these values according to when you run this script.
    dp = DominantProvider({
        'SR00': 'SR609',
        'cu00': 'cu1610',
    })
    # dp = DummyDominantProvider()
    file_base_name = os.path.basename(os.path.splitext(__file__)[0])
    base_folder = safe_get_base_folder(folder=file_base_name)
    config = {
        'base_folder': base_folder,
        'strategy_period': Periodicity.FIVE_MINUTE,
        'instrument_ids': ['cu00', 'SR00'],
        'parameters': {
            'position': 0,
        },
        'logger': StrategyUtils.get_logger(base_folder, has_console=True, has_file=True, append=True),

        # Here we indicate to use remote account, so the strategy will first load the positions from
        # the remote trading account.
        'use_remote_account': True
    }

    # Initialize the strategy.
    account = CtpSimulationAccounts.get_account('simnow_future')
    ctp_factory = CtpRealFactory.create(account=account)
    ctp_factory.set_dominant_provider(dp)

    strategy = ChangePosition(**config)
    strategy.set_ctp_factory(ctp_factory)
    strategy.run()


if __name__ == '__main__':
    main()



```
