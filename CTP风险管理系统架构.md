
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


client_tui.lua
	- 