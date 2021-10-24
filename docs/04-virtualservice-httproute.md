# 04-virtualservice-httproute

httproute 的主要功能：满足 HTTPMatchRequest 条件的流量都被路由到 HTTPRouteDestination，执行重定向（HTTPRedirect）、重写（HTTPRewrite）、重试（HTTPRetry）、故障注入（HTTPFaultInjection）、跨站（CorsPolicy）策略等