# 网络运维工具清单：常用公共 BGP Looking Glass 汇总

> 标签：BGP、LookingGlass、路由排障、AS 路径、路由泄露、网络运维

## 前言

BGP Looking Glass（简称 LG，路由窥视镜）是运营商、IXP 开放的只读路由查询服务，本质是给外部工程师提供一个 “借一双运营商骨干网路由器的眼睛” 去看路由的能力。 BGP 是自治系统间路由协议，不同 Tier1、运营商看到的路由优选路径、下一跳、AS Path 完全不一样。当出现前缀不通、路由泄露、RPKI 失效、跨区域访问异常、新 prefix 发布验证等问题时，我们无法登录对方骨干路由器，Looking Glass 就是最常用的排障手段。

常见支持能力：

- 查询指定前缀的 BGP 路由表、AS Path、Next Hop、Local Pref、Origin
- Ping / Traceroute，从该运营商骨干视角探测目标
- 支持 IPv4 / IPv6，部分节点支持 RPKI 校验

> 重要区分：**运营商 Looking Glass = 单台 / 单个 PoP 路由器视角**；**RouteView / RIPE RIS = 路由收集器**，收集全球大量对等节点路由，适合看路由全网传播情况，二者用途不一样，很多人容易混淆。

> 补充说明：国内三大运营商、多数云厂商**基本不对外开放公网 Looking Glass**，排查国内路由视角，一般只能自建或者对接国内 IXP 路由服务器。

## 一、主流 Tier1 / 运营商公共 Looking Glass（Web）

表格

| 运营商              | ASN  | 访问地址                                      | 简要说明                                                     |
| ------------------- | ---- | --------------------------------------------- | ------------------------------------------------------------ |
| Hurricane Electric  | 6939 | https://lg.he.nethttps://bgp.he.net/super-lg/ | 运维首选，全球多 PoP，界面友好，v4/v6 齐全，ping/traceroute/bgp 都支持 |
| Arelion（原 Telia） | 1299 | https://lg.twelve99.net                       | 欧洲老牌 Tier1，排查欧洲方向路由非常好用                     |
| Lumen（原 Level3）  | 3356 | https://lookingglass.lumen.com                | 北美骨干，北美侧路由验证首选                                 |
| NTT                 | 2914 | https://gin.ntt.net                           | 全球多节点，日韩、亚太方向排查很合适                         |
| Cogent              | 174  | https://www.cogentco.com/en/looking-glass     | 低成本 Transit 运营商，常用于排查低价线路路由问题            |
| Tata                | 6453 | https://lg.as6453.net                         | 亚太、南亚方向视角                                           |
| Zayo                | 6461 | https://lg.zayo.com                           | 北美运营商                                                   |
| RETN                | 9002 | http://lg.retn.net                            | 俄欧方向                                                     |
| PCH                 | 42   | https://www.pch.net/tools/looking_glass       | IXP 收集视角                                                 |

## 二、路由收集器 & 聚合查询工具（看全网路由传播）

这类不是单路由器 LG，是路由收集平台，和大量 AS 建立 BGP 会话，收集 RIB、BGP 更新，适合排查路由是否全网扩散、历史路由变更、劫持溯源。

1. RouteViews
   - `route-views.routeviews.org`（telnet / ssh）
   - `route-views3.routeviews.org`
   - `route-views6.routeviews.org`（IPv6）
   - 使用方式：ssh/telnet 登录，执行 `show ip bgp x.x.x.x/24`，适合批量脚本、MRT 数据获取。
2. RIPEstat（RIPE RIS）
   - https://stat.ripe.net/
   - 亮点：路由历史回溯、路由变更时间线、RPKI 状态、AS 关系，溯源路由泄露 / 劫持神器。
3. bgp.tools
   - https://bgp.tools/
   - 可视化 AS 路径，界面简洁，快速看前缀起源、AS 链路关系，适合快速截图汇报。

## 三、汇总索引站（收藏备用）

- https://www.bgplookingglass.com/：全球 LG 汇总目录，大量小众 IXP、运营商 LG
- https://whois.ipip.net/looking-glass/：IPIP 整理的 Looking Glass 列表

## 四、常用查询命令参考（Cisco 风格，大部分 LG 兼容）

```
# 查看前缀BGP路由（最常用）
show ip bgp 1.1.1.0/24

# 查看IPv6前缀
show bgp ipv6 2400:xxxx::/32

# traceroute探测
traceroute 1.1.1.1

# ping连通性
ping 1.1.1.1
```

> 提示：不同厂商（Juniper、FRR、BIRD）LG 命令语法不一样，部分站点提供下拉表单，不需要手动敲命令。

## 五、典型使用场景

1. 新发布 IP 前缀，验证 Transit/Peer 是否收到路由；
2. 用户反馈某区域访问不通，用不同 Tier1 LG 交叉验证，定位是路由劫持、黑洞、优选路径异常；
3. 路由泄露、BGP 劫持事件溯源，配合 RIPEstat 看路由变更时间；
4. 割接、AS 迁移前，多视角验证路由收敛；
5. 排查 RPKI ROA 配置错误导致路由被拒收。

## 六、局限性提醒

1. Looking Glass 是**只读权限**，无法修改任何 BGP 策略；
2. 只是该路由器 / 收集器的本地 RIB，**不代表全网所有路由器都这么选路**；
3. 部分节点会做限流、屏蔽、过滤，结果仅作参考；
4. 不能替代真实 BGP 探针、实时监控系统（BGPalerter、ExaBGP 等）做告警。

## 结语

BGP Looking Glass 是网络工程师日常排障的轻量利器，建议把 HE、NTT、RIPEstat、bgp.tools 加入书签。遇到跨网路由类故障时，多拿 2~3 个不同 Tier1 视角交叉比对，更容易定位根因。