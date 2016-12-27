---
title: Strategy monitor
layout: post
category: web
language: en
---

This article explains how to use the [strategy monitor](http://www.ctxalgo.com/strategy-monitor) page.

# Url parameters
#### `strategies=STRATEGIES`
The `strategies` parameter is a strategy selector. `STRATEGIES` is a comma-separated list of strategy names. When specified, the current strategy-monitor page only displays the selected strategies. A strategy name can be in form of product.strategy or product. If the latter, all strategies from that product are selected. If not present, all strategies from all products that are monitored by the strategy monitor page are displayed.

For example, the url `/strategy-monitor?strategies=test,test2.strategy1` selects all strategies from the `test` product and the `test2.strategy1` strategy in the `test2` product.


#### `trading-day=TRADING_DAY`
The `trading-day` parameter sets the trading day on which strategy information is displayed. The trading day is in format `YYYYMMDD`. If not present, display information of the current trading day.

For example, the url `/strategy-monitor?trading-day=20161227` specifies the trading day `2016-12-27`.

#### `logs=LOG_TAGS`
This paramater specifies the content pre-entered into the strategy log table (at the bottom of the page). It only has effect if only one strategy is selected in current page. This parameter filters strategy logs.

For example, `/strategy-monitor?strategies=test.s1&logs=WARNING`, selects a single strategy `test.s1` and enter `WARNING` into the strategy log table.

#### `log-limit=LOG_LIMIT`
This parameter specifies the maximal number of messages that the strategy logs table is allowed to display before it enters partial display mode. `LOG_LIMIT` is an integer, when not present, defaults to 10000. A strategy can send millions of messages to the strategy monitor, but rendering all those messages on the website may causing performance problems. When the number of messages exceeds `LOG_LIMIT`, the strategy log table will enter partical display mode, which only displays essential messages, such as `ORDER`, `TRADED`. All messages are still available from the message download link.
