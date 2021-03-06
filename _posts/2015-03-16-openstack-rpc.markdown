---
layout: post
title:  "Openstack RPC"
date:   2015-03-16 12:08:57
categories: openstack
---

### RabbitMQ的消息模型

RabbitMQ 基本概念:

* Producer: 消息生产者
* Consumer: 消息消费者
* Exchange: 接受消息的载体
* Message: RabbitMQ中的基本消息单元
* Queue: 存储并接受Exchange分发的消息, 用于Consumer接受消息
* Routing Key: 标识Message, 用于Exchange分发消息的依据
* Binding: Queue与Exchange绑定, 可选的binding key参数用于指定特定的消息(相当于消息的routing key)

在RabbitMQ中, 消息是发送给`Exchange`的, 而不是`Queue`, `Producer`将带有`Routing Key`的消息发送给相应的
`Exchange`后, 整个消息的发送过程就结束了; 消息的接收是通过`Queue`完成的, `Consumer`在RabbitMQ中定义
一个`Queue`, 与相应的`Exchange`进行`Binding`, 在`Binding`时可指定感兴趣的消息, 通过`binding key`来完成,这
样消息就会被`Exchange`根据`binding key`分发到相应的`Queue`上,  `Consumer`接受`Queue`的消息即可, 
这样整个消息的接收就完成了.

![img](http://www.rabbitmq.com/img/tutorials/python-three-overall.png){: .center-image}

如上图所示, `P`代表`Producer`, `x`代表`Exchange`, 红色的代表`Queue`以及相应的queue name, binding表示上
述的`Binding`过程, `C`代表`Consumer`.

### RabbitMQ中的Exchange

Exchange在RabbitMQ中也有相应的分类:`Direct` `Fanout` `Topic`

Exchange的分类是为了实现不同的通信模式

* Fanout: 消息将分发给所有与`Exchange`绑定的`Queue`
* Direct: 消息将通过 *完全匹配* `Routing Key`分发到相应的`Queue`
* Topic: 消息将通过 *正则匹配(支持特殊字符)* `Routing Key`分发到相应的`Queue`

    `*`匹配单个字符<br>
    `#`匹配0或多个字符

因此, 当使用`Fanout`类型的`Exchange`, 消息将分发至所有与该`Exchange`绑定的`Queue`上, 此时不会匹配`Routing Key`, 
当使用`Direct`类型的`Exchange`, 消息将通过*完全匹配*(`Routing Key`与`Binding Key`)被分发至相应的`Queue`, 同理, 
`Topic`类型的`Exchange`会进行*正则匹配*.

如图, `Direct Exchange`类型的消息模型:
![img](http://www.rabbitmq.com/img/tutorials/direct-exchange.png){: .center-image}

如图, `Topic Exchange`类型的消息模型:
![img](http://www.rabbitmq.com/img/tutorials/python-five.png){: .center-image}

从两种类型的`Exchange`可以看出, 区别仅在于分发消息时对于`Routing Key`的匹配方式, `Direct`是*完全匹配*`Queue`的`Binding Key`, 
`Topic`则是使用*正则匹配*.

### Openstack RPC

在Openstack中, 组件内的通信主要通过消息队列完成, 这样可以使整个系统以松耦合的方式进行协作, 以RabbitMQ为例,
Openstack实现了组件间的RPC调用, 主要包括*同步*和*异步*两种调用方式: 

{% highlight python linenos %}
    """oslo/messaging/rpc/client.py"""
    class RPCClient(object):

        """A class for invoking methods on remote servers.
        The RPCClient class is responsible for sending method invocations to remote
        servers via a messaging transport.

        A cast() invocation just sends the request and returns immediately. 
        A call() invocation waits for the server to send a return value.
        """

        def cast(self, ctxt, method, **kwargs):
            """Invoke a method and return immediately.
            """
            self.prepare().cast(ctxt, method, **kwargs)
        def call(self, ctxt, method, **kwargs):
            """Invoke a method and wait for a reply.
            """
{% endhighlight %}

插入的代码只是RPCClient实现的一部分, 从注释可以看出, `cast函数`属于异步调用, `call函数`属于同步调用.

### cast实现方式
![img](http://docs.openstack.org/developer/nova/_images/flow2.png){: .center-image}

1. rpc调用方会初始化`Topic Publisher`, 将消息以`Routing Key`为topic发送至配置的`Exchange`中
2. rpc接收方会初始化两个`Topic Cosumer`, 分别通过topic和topic.host为`binding key`将`Queue`与`Exchange`绑定, 
这样接收方就可以接收这两种类型的rpc message

    `topic`即为消息的`Routing Key`, 所以接收方可以接受到调用方的rpc message.<br>
    `topic.host`为特定的host上的接收方获取rpc message的方式, 例如: l3-agent.host1(通知host1上的l3-agent进行相关rpc调用)

### call实现方式
![img](http://docs.openstack.org/developer/nova/_images/flow1.png){: .center-image}

1. rpc调用方初始化`Topic Publisher`用于发送rpc message, 同时初始化一个`Direct Consumer`用于接收返回的rpc调用结果, 
为了保证收到相应的rpc调用的结果, rpc message中会保存一个唯一标识该消息的messga id(UUID类型), 消息以`Routing Key`为topic发送至
`Exchange`中
2. rpc接收方会初始化两个`Topic Cosumer`, 分别通过topic和topic.host为`binding key`将`Queue`与`Exchange`绑定, rpc
接收方执行rpc调用, 完成后通过`Direct Publisher`将结果发送到消息队列中, 返回的执行结果message的`Routing Key`为message id
(唯一的UUID), 将被发送至`Exchange`(名称同样是唯一的message id), 那样调用方的`Direct Consumer`就可以收到rpc调用的结果了.

### 以ovs_neutron_agent为例理解rpc的实现

在ovs_neutron_agent的代码中, rpc的初始化如下:
{% highlight python linenos %}

    """plugins/openvswitch/agent/ovs_neutron_agent.py"""
    def setup_rpc(self):

        ......
        #作为rpc调用方的初始化
        self.plugin_rpc = OVSPluginApi(topics.PLUGIN)   #与ml2 plgin通信
        self.state_rpc = agent_rpc.PluginReportStateAPI(topics.PLUGIN)

        #作为rpc接收方的初始化
        self.topic = topics.AGENT
        # Handle updates from service
        self.endpoints = [self]
        # Define the listening consumers for the agent
        consumers = [[topics.PORT, topics.UPDATE],
                     [topics.NETWORK, topics.DELETE],
                     [constants.TUNNEL, topics.UPDATE],
                     [topics.SECURITY_GROUP, topics.UPDATE],
                     [topics.DVR, topics.UPDATE]]
        if self.l2_pop:
            consumers.append([topics.L2POPULATION,
                              topics.UPDATE, cfg.CONF.host])
        self.connection = agent_rpc.create_consumers(self.endpoints,
                                                     self.topic,
                                                     consumers)
        ......

{% endhighlight %}

self.topic为ovs_neutron_agent会接收的rpc message的类型前缀, 在topics.py中定义:
{% highlight python linenos %}
    """common/topics.py"""
    AGENT = 'q-agent-notifier'
{% endhighlight %}
在neutron中, 消息可以分为不同的类型, 以AGENT='q-agent-notifier'为例, 表示为通知agent端的消息类型.

self.consumers为ovs_neutron_agent会接收的rpc message的具体类型, 在agent_rpc.create_consumers()函数中, 完成了consumers的初始化:
{% highlight python linenos %}

    """agent/rpc.py"""
    def create_consumers(endpoints, prefix, topic_details):
        """
        :param endpoints: The list of endpoints to process the incoming messages.
        :param prefix: Common prefix for the plugin/agent message queues.
        :param topic_details: A list of topics. Each topic has a name, an
                              operation, and an optional host param keying the
                              subscription to topic.host for plugin calls.
        """

        connection = n_rpc.create_connection(new=True)
        for details in topic_details:
            topic, operation, node_name = itertools.islice(
                itertools.chain(details, [None]), 3)

            topic_name = topics.get_topic_name(prefix, topic, operation)
            connection.create_consumer(topic_name, endpoints, fanout=True)
            if node_name: #node_name即为host name
                node_topic_name = '%s.%s' % (topic_name, node_name)
                connection.create_consumer(node_topic_name,
                                           endpoints,
                                           fanout=False)
        connection.consume_in_threads()
        return connection
{% endhighlight %}

这里需要关注的就是create\_consumer函数, 以及传递给该函数的参数topic_name, endpoints, 对于fanout参数, 当
host存在时, 就不需要fanout类型, 因为只需要特定的host接收. topic\_name为prefix + topic + operation, 例如:
`q-agent-notifier-port-update`, `q-agent-notifier-port-update.hostname`, `q-agent-notifier-security_grout-update`等, 
这些例子就会作为topic_name传递给create_consumer函数, endpoints为执行rpc函数的载体, 此处应该传递的:

{% highlight python linenos %}
        self.endpoints = [self]  #上面的setup_rpc函数中, 即为实例本身
{% endhighlight %}

{% highlight python linenos %}

    def get_server(target, endpoints, serializer=None):
        assert TRANSPORT is not None
        serializer = RequestContextSerializer(serializer)
        return messaging.get_rpc_server(TRANSPORT, target, endpoints,
                                        'eventlet', serializer)

    class Connection(object):

        def __init__(self):
            super(Connection, self).__init__()
            self.servers = []

        def create_consumer(self, topic, endpoints, fanout=False):
            target = messaging.Target(
                topic=topic, server=cfg.CONF.host, fanout=fanout)
            server = get_server(target, endpoints)
            self.servers.append(server)

        def consume_in_threads(self):
            for server in self.servers:
                server.start()
            return self.servers

    # functions
    def create_connection(new=True):
        return Connection()
{% endhighlight %}

在create_consumer函数中, 首先初始化target, 其作用是封装了关于rpc message相关联的属性, 
比如该消息发送到的`Exchange`, 消息的topic(Routing Key)等.

> 注意: 在neutron中, 消息的topic其实就是消息发送进`Exchange`所带有的`Routing Key`, 比如以topic `q-agent-notifier-port-update`
> 为例, 与port-update相关的rpc message应该都是`Routing Key`为`q-agent-notifier-port-update`的.

然后在get_server函数中, serializer为序列化rpc消息的函数, 用于将raw rpc message序列化为适合neutron上下文的dict, 可以不用
深入去看, 明白其含义即可, TRANSPORT为消息的传递载体, 因为openstack中可以使用rabbitmq, 以及zeromq等消息队列, TRANSPOST主要
完成了对相应消息队列功能的封装, 这儿以rabbitmq为消息队列, 也就是说, 消息是通过TRANSPORT(rabbimq)传递的.

{% highlight python linenos %}

    def get_rpc_server(transport, target, endpoints,
                       executor='blocking', serializer=None):
        """Construct an RPC server.

        The executor parameter controls how incoming messages will be received and
        dispatched. By default, the most simple executor is used - the blocking
        executor.

        :param transport: the messaging transport
        :type transport: Transport
        :param target: the exchange, topic and server to listen on
        :type target: Target
        :param endpoints: a list of endpoint objects
        :type endpoints: list
        :param executor: name of a message executor - for example
                         'eventlet', 'blocking'
        :type executor: str
        :param serializer: an optional entity serializer
        :type serializer: Serializer
        """
        dispatcher = rpc_dispatcher.RPCDispatcher(target, endpoints, serializer)
        return msg_server.MessageHandlingServer(transport, dispatcher, executor)
{% endhighlight %}

在messaging.get_rpc_server函数中, 进行了dispatcher的初始化, 它的主要作用为从TRANSPORT中接受感兴趣的消息, 
而target描述了这些感兴趣的消息的属性, target记录了这些信息.

{% highlight python linenos %}

    class MessageHandlingServer(object):
        """Server for handling messages.

        Connect a transport to a dispatcher that knows how to process the
        message using an executor that knows how the app wants to create
        new tasks.
        """

        def __init__(self, transport, dispatcher, executor='blocking'):
            """Construct a message handling server.

            The dispatcher parameter is a callable which is invoked with context
            and message dictionaries each time a message is received.

            The executor parameter controls how incoming messages will be received
            and dispatched. By default, the most simple executor is used - the
            blocking executor.

            :param transport: the messaging transport
            :type transport: Transport
            :param dispatcher: a callable which is invoked for each method
            :type dispatcher: callable
            :param executor: name of message executor - for example
                             'eventlet', 'blocking'
            :type executor: str
            """
            self.conf = transport.conf

            self.transport = transport
            self.dispatcher = dispatcher
            self.executor = executor

            try:
                mgr = driver.DriverManager('oslo.messaging.executors',
                                           self.executor)
            except RuntimeError as ex:
                raise ExecutorLoadFailure(self.executor, ex)
            else:
                self._executor_cls = mgr.driver
                self._executor = None

            super(MessageHandlingServer, self).__init__()

        def start(self):
            """Start handling incoming messages.
            """
            if self._executor is not None:
                return
            try:
                listener = self.dispatcher._listen(self.transport)
            except driver_base.TransportDriverError as ex:
                raise ServerListenError(self.target, ex)

            self._executor = self._executor_cls(self.conf, listener,
                                                self.dispatcher)
            self._executor.start()

{% endhighlight %}

接下来在MessageHandlingServer(transport, dispatcher, executor)中, executor为'blocking', 表示rpc函数的执行会
阻塞当前线程, 相应的参数还有'eventlet', 表示在新的eventlet线程中执行, 我们也不需要关心这部分内容, 但是
需要明白其作用. 在上面的start函数中, listener为dispatcher的_listen所返回的结果, 从listener中, 我们就可以
接收到一个一个的消息了, 然后通过self._executor.start(), 开始获取消息并进行相应的rpc调用.

{% highlight python linenos %}

    def spawn_with(ctxt, pool):
        """This is the equivalent of a with statement
        but with the content of the BLOCK statement executed
        into a greenthread

        exception path grab from:
        http://www.python.org/dev/peps/pep-0343/
        """

        def complete(thread, exit):
            exc = True
            try:
                try:
                    thread.wait()
                except Exception:
                    exc = False
                    if not exit(*sys.exc_info()):
                        raise
            finally:
                if exc:
                    exit(None, None, None)

        callback = ctxt.__enter__()
        thread = pool.spawn(callback)
        thread.link(complete, ctxt.__exit__)

        return thread

    class EventletExecutor(base.ExecutorBase):

        """A message executor which integrates with eventlet.

        This is an executor which polls for incoming messages from a greenthread
        and dispatches each message in its own greenthread.

        The stop() method kills the message polling greenthread and the wait()
        method waits for all message dispatch greenthreads to complete.
        """

        def __init__(self, conf, listener, dispatcher):
            super(EventletExecutor, self).__init__(conf, listener, dispatcher)
            self.conf.register_opts(_eventlet_opts)
            self._thread = None
            self._greenpool = greenpool.GreenPool(self.conf.rpc_thread_pool_size)
            self._running = False

        def start(self):
            if self._thread is not None:
                return

            @excutils.forever_retry_uncaught_exceptions
            def _executor_thread():
                try:
                    while self._running:
                        incoming = self.listener.poll(timeout=base.POLL_TIMEOUT)
                        if incoming is not None:
                            spawn_with(ctxt=self.dispatcher(incoming),
                                       pool=self._greenpool)
                except greenlet.GreenletExit:
                    return

            self._running = True
            self._thread = eventlet.spawn(_executor_thread)
{% endhighlight %}

在self.\_executor的实现中, start函数的self.listener.poll()函数不断获取rpc消息, 并将新消息在新的eventlet thread中执行, 其中spwan_with中会产生
新的thread, 同时传递给spwan\_with的参数为ctxt=self.dispatcher(incoming), 需要特别注意, 因为dispatcher实现了[\_\_call\_\_][1]方法, 由于
self.dispatcher是一个可调用对象, 因此ctxt=self.dispatcher(incoming)这样的调用是可行的.

现在我们就可以看看在dispatcher中, 消息是怎样得到处理的.

{% highlight python linenos %}

    class RPCDispatcher(object):
        """A message dispatcher which understands RPC messages.

        A MessageHandlingServer is constructed by passing a callable dispatcher
        which is invoked with context and message dictionaries each time a message
        is received.

        RPCDispatcher is one such dispatcher which understands the format of RPC
        messages. The dispatcher looks at the namespace, version and method values
        in the message and matches those against a list of available endpoints.

        Endpoints may have a target attribute describing the namespace and version
        of the methods exposed by that object. All public methods on an endpoint
        object are remotely invokable by clients.
        """

        def __init__(self, target, endpoints, serializer):
            """Construct a rpc server dispatcher.

            :param target: the exchange, topic and server to listen on
            :type target: Target
            """
    
            self.endpoints = endpoints
            self.serializer = serializer or msg_serializer.NoOpSerializer()
            self._default_target = msg_target.Target()
            self._target = target
    
        def _listen(self, transport):
            return transport._listen(self._target)
    
        @staticmethod
        def _is_namespace(target, namespace):
            return namespace == target.namespace
    
        @staticmethod
        def _is_compatible(target, version):
            endpoint_version = target.version or '1.0'
            return utils.version_is_compatible(endpoint_version, version)
    
        def _do_dispatch(self, endpoint, method, ctxt, args):
            ctxt = self.serializer.deserialize_context(ctxt)
            new_args = dict()
            for argname, arg in six.iteritems(args):
                new_args[argname] = self.serializer.deserialize_entity(ctxt, arg)
            result = getattr(endpoint, method)(ctxt, **new_args)
            return self.serializer.serialize_entity(ctxt, result)
    
        @contextlib.contextmanager
        def __call__(self, incoming):
            incoming.acknowledge()
            yield lambda: self._dispatch_and_reply(incoming)
    
        def _dispatch_and_reply(self, incoming):
            try:
                incoming.reply(self._dispatch(incoming.ctxt,
                                              incoming.message))
            except ExpectedException as e:
                LOG.debug(u'Expected exception during message handling (%s)',
                          e.exc_info[1])
                incoming.reply(failure=e.exc_info, log_failure=False)
            except Exception as e:
                # sys.exc_info() is deleted by LOG.exception().
                exc_info = sys.exc_info()
                LOG.error(_('Exception during message handling: %s'), e,
                          exc_info=exc_info)
                incoming.reply(failure=exc_info)
                # NOTE(dhellmann): Remove circular object reference
                # between the current stack frame and the traceback in
                # exc_info.
                del exc_info

        def _dispatch(self, ctxt, message):
            """Dispatch an RPC message to the appropriate endpoint method.

            :param ctxt: the request context
            :type ctxt: dict
            :param message: the message payload
            :type message: dict
            :raises: NoSuchMethod, UnsupportedVersion
            """
            method = message.get('method')
            args = message.get('args', {})
            namespace = message.get('namespace')
            version = message.get('version', '1.0')

            found_compatible = False
            for endpoint in self.endpoints:
                target = getattr(endpoint, 'target', None)
                if not target:
                    target = self._default_target

                if not (self._is_namespace(target, namespace) and
                        self._is_compatible(target, version)):
                    continue

                if hasattr(endpoint, method): #从endpoint中获取相应的rpc method
                    localcontext.set_local_context(ctxt) #执行rpc调用
                    try:
                        return self._do_dispatch(endpoint, method, ctxt, args) #返回序列化的结果
                    finally:
                        localcontext.clear_local_context()

                found_compatible = True

            if found_compatible:
                raise NoSuchMethod(method)
            else:
                raise UnsupportedVersion(version, method=method)
{% endhighlight %}

在\_\_call\_\_方法中, 返回的[lambda][2]即为: self._dispatch_and_reply(incoming), 然后消息会在\_dispatch函数中被处理并返回相应的
rpc调用的结果.

### Transport如何通过Target获取相应的消息

{% highlight python linenos %}
        def _listen(self, transport):
            return transport._listen(self._target)
{% endhighlight %}

在上面dispatcher的实现中, 可以看到_listen函数是通过tranport.\_listen实现的, 并相应的传递了self._target作为参数.
因此, 我们可以通过理解Transport的实现来理解openstack neutron中topic的含义.

{% highlight python linenos %}

    class Transport(object):
        """A messaging transport.
        This is a mostly opaque handle for an underlying messaging transport
        driver.
        It has a single 'conf' property which is the cfg.ConfigOpts instance used
        to construct the transport object.
        """
        def __init__(self, driver):
            self.conf = driver.conf
            self._driver = driver

        def _send(self, target, ctxt, message, wait_for_reply=None, timeout=None,
                  retry=None):
            if not target.topic:
                raise exceptions.InvalidTarget('A topic is required to send',
                                               target)
            return self._driver.send(target, ctxt, message,
                                     wait_for_reply=wait_for_reply,
                                     timeout=timeout, retry=retry)

        def _send_notification(self, target, ctxt, message, version, retry=None):
            if not target.topic:
                raise exceptions.InvalidTarget('A topic is required to send',
                                               target)
            self._driver.send_notification(target, ctxt, message, version,
                                           retry=retry)

        def _listen(self, target):
            if not (target.topic and target.server):
                raise exceptions.InvalidTarget('A server\'s target must have '
                                               'topic and server names specified',
                                               target)
            return self._driver.listen(target) #相应的实现在driver中实现
{% endhighlight %}

可以看到, 不同的transport都实现了listen函数, 以rabbitmq的实现为例, driver代码如下:

{% highlight python linenos %}

    class AMQPDriverBase(base.BaseDriver):
    
        def __init__(self, conf, url, connection_pool,
                     default_exchange=None, allowed_remote_exmods=None):
            super(AMQPDriverBase, self).__init__(conf, url, default_exchange,
                                                 allowed_remote_exmods)
    
            self._default_exchange = default_exchange
    
            self._connection_pool = connection_pool
    
            self._reply_q_lock = threading.Lock()
            self._reply_q = None
            self._reply_q_conn = None
            self._waiter = None
    
        def _get_exchange(self, target):
            return target.exchange or self._default_exchange
    
        def _get_connection(self, pooled=True):
            return rpc_amqp.ConnectionContext(self._connection_pool,
                                              pooled=pooled)
    
        def _get_reply_q(self):
            with self._reply_q_lock:
                if self._reply_q is not None:
                    return self._reply_q
    
                reply_q = 'reply_' + uuid.uuid4().hex
    
                conn = self._get_connection(pooled=False)
    
                self._waiter = ReplyWaiter(self.conf, reply_q, conn,
                                           self._allowed_remote_exmods)
    
                self._reply_q = reply_q
                self._reply_q_conn = conn
    
            return self._reply_q
    
        def _send(self, target, ctxt, message,
                  wait_for_reply=None, timeout=None,
                  envelope=True, notify=False, retry=None):
    
            # FIXME(markmc): remove this temporary hack
            class Context(object):
                def __init__(self, d):
                    self.d = d
    
                def to_dict(self):
                    return self.d
    
            context = Context(ctxt)
            msg = message
    
            if wait_for_reply:
                msg_id = uuid.uuid4().hex
                msg.update({'_msg_id': msg_id})
                LOG.debug('MSG_ID is %s', msg_id)
                msg.update({'_reply_q': self._get_reply_q()})
    
            rpc_amqp._add_unique_id(msg)
            rpc_amqp.pack_context(msg, context)
    
            if envelope:
                msg = rpc_common.serialize_msg(msg)
    
            if wait_for_reply:
                self._waiter.listen(msg_id)
    
            try:
                with self._get_connection() as conn:
                    if notify:
                        conn.notify_send(self._get_exchange(target),
                                         target.topic, msg, retry=retry)
                    elif target.fanout:
                        conn.fanout_send(target.topic, msg, retry=retry)
                    else:
                        topic = target.topic
                        if target.server:
                            topic = '%s.%s' % (target.topic, target.server)
                        conn.topic_send(exchange_name=self._get_exchange(target),
                                        topic=topic, msg=msg, timeout=timeout,
                                        retry=retry)
    
                if wait_for_reply:
                    result = self._waiter.wait(msg_id, timeout)
                    if isinstance(result, Exception):
                        raise result
                    return result
            finally:
                if wait_for_reply:
                    self._waiter.unlisten(msg_id)
    
        def send(self, target, ctxt, message, wait_for_reply=None, timeout=None,
                 retry=None):
            return self._send(target, ctxt, message, wait_for_reply, timeout,
                              retry=retry)
    
        def send_notification(self, target, ctxt, message, version, retry=None):
            return self._send(target, ctxt, message,
                              envelope=(version == 2.0), notify=True, retry=retry)
    
        def listen(self, target):
            conn = self._get_connection(pooled=False)
    
            listener = AMQPListener(self, conn)
    
            conn.declare_topic_consumer(exchange_name=self._get_exchange(target), #定义Topic Consumer
                                        topic=target.topic,
                                        callback=listener)
            conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                        topic='%s.%s' % (target.topic,
                                                         target.server),
                                        callback=listener)
            conn.declare_fanout_consumer(target.topic, listener)
    
            return listener
    
        def listen_for_notifications(self, targets_and_priorities, pool):
            conn = self._get_connection(pooled=False)
    
            listener = AMQPListener(self, conn)
            for target, priority in targets_and_priorities:
                conn.declare_topic_consumer(
                    exchange_name=self._get_exchange(target),
                    topic='%s.%s' % (target.topic, priority),
                    callback=listener, queue_name=pool)
            return listener
{% endhighlight %}

从上面listen的代码, 就回到了rpc call和rpc cast模型, 定义Topic Consumer, 但是为什么会初始化3个`Consumer`呢？

这要从rpc方法调用的方式说起, rpc call和 rpc cast有两种方式: topic + topic.host, rpc fanout_cast有一种方式: fanout, 所以对应的`Consumer`
也有3种, 因为在消息的接收方看来, 我不需要关心消息是以什么方式传输过来的, 无论是rpc call, 或者rpc cast, 或rpc fanout_cast, 对于
消息本身才是它需要关心的, 因此它可以接收三种方式发送过来的消息, 所以会初始化3种`Consumer`.

### 总结

结合RabbbitMQ的基本概念, 不难看出, openstack中与rpc调用相关的topic, 对消息发送方(rpc调用方)来说, 它就是
消息的`Routing Key`, 对于消息接收方(rpc接受方), 它就是初始化`Consumer`时用于接受消息的`Queue`的名称, 
且`Queue`与`Exchange`的Binding Key也为它.

### 参考网站

RabbitMQ:   <http://www.rabbitmq.com/>

[1]: <http://stackoverflow.com/questions/5824881/python-call-special-method-practical-example>

[2]: <http://www.secnetix.de/olli/Python/lambda_functions.hawk>
