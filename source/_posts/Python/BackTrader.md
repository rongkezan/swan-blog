---
title: Selenium
date: {{ date }}
categories:
- Python
---

> https://www.backtrader.com/docu/

## BackTrader 架构

- DataFeed：数据传递
- Cerebro：大脑，核心引擎
- Broker：经纪人，负责 订单执行、仓位管理、交易费率设置
- Startegy：策略类
  - Observer：观察模块，策略执行
  - Analyzer：分析模块，策略统计
  - Indicator：指标模块
- Sizer：开仓、平仓的的计算和执行

<img src="https://img-blog.csdnimg.cn/b64b4dc71bab4285a8996200c1166291.png" alt="在这里插入图片描述" style="zoom: 33%;" />

## BackTrader Demo Code

```python
import pandas as pd
import backtrader as bt


class TestStrategy(bt.Strategy):

    def __init__(self):
        self.sma = bt.indicators.MovingAverageSimple()
        pass

    def next(self):
        data = self.datas[0]
        if not self.position:
						# 以开盘价限价下单
	  				self.buy(exectype=bt.Order.Limit, price=data.open[0], size=1)
        else:
            self.close(size=1)


if __name__ == '__main__':
    data_frame = pd.read_csv("demo.csv")
    cerebro = bt.Cerebro()
    data_frame['datetime'] = pd.to_datetime(data_frame['datetime'])
    data_frame.set_index('datetime', inplace=True)
    data_feeds_1h = bt.feeds.PandasData(dataname=data_frame)
    # 数据
    cerebro.adddata(data_feeds_1h)
    # 策略
    cerebro.addstrategy(TestStrategy)
    # 经纪人
    cerebro.broker.setcash(1000000)
    # 手续费
    cerebro.broker.setcommission(0.0002)
    # 执行回测
    result = cerebro.run()
    post_cash = cerebro.broker.getcash()
    cerebro.plot(style='candle')
```

