# ai_for_trading



trader: "User/Trader"

edge: Edge

star: STAR

inav: "INav/Quartz" {
  pricer: "Quant Pricer"
  inavPublisher: inavPublisher
}
edge -> equityForce: sends orders to
mdFeedHanders: "Market Data Feed-Handers" {
  btecFeed: "BrokerTec Feeds"
  bigFeeds: "Equity BIG-FEEDS"
}
# replica 1 <-> replica 2
# a -> b: To err is human, to moo bovine {
#   source-arrowhead: 1
#   target-arrowhead: * {
#     shape: diamond
#   }
# }

forces: "Force Instances" {
  fiForce: FI Force62411
  equityForce: Equity Force
}
phoenix: Phoenix
forces <- phoenix
mdFeedHanders -> edge.megaBook: via LQ2
edge: {
  shm: "The Shared Memory (SHM)" {
    orderBook: "Order Book"
    allOtherParameters: "all others parameters"
  }
  megaBook: "MegaBook (MD Subscriber and OrderBook Builder)"
  inavSubscriber: INav Data Subsciber
  xti: XTI
  relay: Gateways : {
    forceConnectors: "Force Connectors (Relay's)"
    fixlinkConnector: "FixLink Connector"
  }
}
inav.inavPublisher -> edge.inavSubscriber: via LQ2
inav -> edge.inavSubscriber
edge.megaBook -> edge.shm.orderBook
edge.xti -> forces: (optional) order instructions via FOP
fiForce -> star: sends executions to
equityForce -> star: sends executions to
