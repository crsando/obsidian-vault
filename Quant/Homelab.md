

家里机器的分工
- ironbox：8G GPU跑Whisper + Pytorch + 跑Quant服务
	- TickData的长期持久化保存（硬盘最大）
- thinklab（T460s）：开发机，tailscale exit node，openclaw
- mbp12（Macbook Pro 12）：QuestDB数据库 + 中转服务
- mba1：Alma + Openclaw（梅梅的）

需要解决的问题：
- 百度网盘下载 + 转存nas
- TickData的长期持久化保存