---
layout:	post
title:	"Openstack DHCP"
date:	2015-03-19
categories:	openstack
---

##Openstack DHCP

基本介绍:
dhcp agent负责ip分配，主要包括DhcpAgent, DhcpAgentWithStateReport和DhcpPluginApi类。
主函数如下:
{% highlight python linenos %}
	"""neutron/agent/dhcp_agent.py"""
	def main():
	    register_options()
	    common_config.init(sys.argv[1:])
	    config.setup_logging(cfg.CONF)
	    server = neutron_service.Service.create(
	        binary='neutron-dhcp-agent',
	        topic=topics.DHCP_AGENT,
	        report_interval=cfg.CONF.AGENT.report_interval,
	        manager='neutron.agent.dhcp_agent.DhcpAgentWithStateReport')
	    service.launch(server).wait()
{% endhighlight %}
注册配置之后初始化了log，然后初始化server并启动运行agent。

dhcp server初始化结构图:

![img](/asset/2015-03-19-openstack-dhcp/dhcp_agent_create_server.png)

neutron server与dhcp agent的交互:

![img](/asset/2015-03-19-openstack-dhcp/dhcp_agent_neutron_remote_invoke.png)

该示意图表示dhcp agent与neutron的远程方法调用的方向,这里的调用都走队列（如rabbitmq），
这些类中DhcpPluginApi生成plugin_rpc实例，DhcpAgent被DhcpAgentWithStateReport继承。
