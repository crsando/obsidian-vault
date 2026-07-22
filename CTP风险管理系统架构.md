
- services/
	- root.lua
	- gateway.lua
	- ctp_trader.lua
		- 只读，只OnRtnTrade和QueryAccount, QueryPositions
	- risk_monitor.lua
		- 监听
			- 新增交易，新增头寸
			- 当前头寸，VaR
			- 当前标的的行情信息
		- 信号预警
			- 行情剧烈波动
			- VCP Pattern

server.ts
	- 提供一个监控面板



Key Component
- 哪些数据需要监控，如何监控


client_tui.lua
	- 连接gateway，发指令
	- 给AI调用，查询当前持仓的信息等