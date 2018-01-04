---
layout: post
title:  "libevent sample HelloWorld.c"
date:   2018-01-03 22:45:10 -0800
categories: libevent
---


1. 创建一个event_base
2. 调用函数evconnlistener_new_bind函数，返回1个listener
3. 调用evsignal_new，增加一个信号事件处理函数
4. 调用event\_base\_dispatch进入事件主循环

<pre>
<code>
struct evconnlistener *
evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb,
    void *ptr, unsigned flags, int backlog, const struct sockaddr *sa,
    int socklen)
</code>
1. 创建套接字和evconnlistener
2. 调用setsockopt,设置套接字选项为SO_KEEPALIVE，根据参数flag再针对性的设定option
3. 调用函数bind对创建的套接字和传入的参数sa进行绑定
4. 调用函数evconnlistener_new
5. return evconnlistener
</pre>

<pre>
<code>
struct evconnlistener *
evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd)
</code>
1. 调用函数listen，将主动套接字转变为被动套接字，backlog可以根据参数设定，如果backlog是小于0的，那么默认是设置成128
2. 分配一个evconnlistener_event，指针名为lev，并且对其进行赋值
	1. 设置lev->base.ops为evconnlistener_event_ops
	2. 设置callback函数lev->base.cb
	3. 设置......
3. 调用函数event_assign，初始化lev->listener(这是一个event)
4. 如果传入的flag参数的LEV_OPT_DISABLE没有被置位的话，调用函数lev->base.ops.enable
==================================================================================
<code>
evconnlistener_ops是一个结构体，里面存放的是函数指针，
struct evconnlistener_ops {
	int (*enable)(struct evconnlistener *);
	int (*disable)(struct evconnlistener *);
	void (*destroy)(struct evconnlistener *);
	void (*shutdown)(struct evconnlistener *);
	evutil_socket_t (*getfd)(struct evconnlistener *);
	struct event_base *(*getbase)(struct evconnlistener *);
};

evconnlistener_event_ops的类型是evconnlistener_ops：
static const struct evconnlistener_ops evconnlistener_event_ops = {
	event_listener_enable,
	event_listener_disable,
	event_listener_destroy,
	NULL, /* shutdown */
	event_listener_getfd,
	event_listener_getbase
};

那么此处调用的lev->base.ops.enable也就是event_listener_enable，这个函数功能是调用函数event_add将lev->listener事件插入添加到事件队列中去。


</code>
</pre>
