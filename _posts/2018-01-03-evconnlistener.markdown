---
layout: post
title:  "[libevent 源码剖析] eventlistener"
date:   2018-01-03 22:45:10 -0800
categories: libevent
---

<font face="微软雅黑">

## eventlistener

```
struct evconnlistener *
evconnlistener_new(
	struct event_base *base,
	evconnlistener_cb cb,
	void *ptr,
	unsigned flags,
	int backlog,
	evutil_socket_t fd)
```
1. 调用listen，将fd变为被动套接字
2. 分配evconnlistener_event ---->lev

_ _ _

```
struct evconnlistener {
    sconst struct evconnlistener_ops *ops；
    void *lock;
    evconnlistener_cb cb;
    evconnlistener_errorcb errorcb;
    void *user_data;
    unsigned flags;
    short refcnt;
    int accept4_flags;
    unsigned enabled : 1;
    };
```
```
 struct evconnlistener_event {
 	stuct evconnlistener base;
 	struct event listener;
 };
```

3. 设置lev->base的ops和cb等

evconnlinstener\_cb是函数指针，evconnnlinstener_new的一个参数传入，作为linsten套接字可读时的callback
```
/**
   A callback that we invoke when a listener has a new connection.
   @param listener The evconnlistener
   @param fd The new file descriptor
   @param addr The source address of the connection
   @param socklen The length of addr
   @param user_arg the pointer passed to evconnlistener_new()
 */
typedef void (*evconnlistener_cb)(struct evconnlistener *, evutil_socket_t, struct sockaddr *, int socklen, void *);
```

ops := evconnlinstener_ops，是一个结构体，结构体成员是函数指针
```
struct evconnlistener_ops {
	int (*enable)(struct evconnlistener *);
	int (*disable)(struct evconnlistener *);
	void (*destroy)(struct evconnlistener *);
	void (*shutdown)(struct evconnlistener *);
	evutil_socket_t (*getfd)(struct evconnlistener *);
	struct event_base *(*getbase)(struct evconnlistener *);
};
```

将ops设定为:
```
static const struct evconnlistener_ops evconnlistener_event_ops = {
	event_listener_enable,
	event_listener_disable,
	event_listener_destroy,
	NULL, /* shutdown */
	event_listener_getfd,
	event_listener_getbase
};
```

4. 调用函数event_assign, 初始化event
```
int
event_assign(struct event *ev, struct event_base *base, evutil_socket_t fd, short events, void (*callback)(evutil_socket_t, short, void *), void *arg)
{
	if (!base)
		base = current_base;
	if (arg == &event_self_cbarg_ptr_)
		arg = ev;

	event_debug_assert_not_added_(ev);

	ev->ev_base = base;

	ev->ev_callback = callback;
	ev->ev_arg = arg;
	ev->ev_fd = fd;
	ev->ev_events = events;
	ev->ev_res = 0;
	ev->ev_flags = EVLIST_INIT;
	ev->ev_ncalls = 0;
	ev->ev_pncalls = NULL;

	if (events & EV_SIGNAL) {
		if ((events & (EV_READ|EV_WRITE|EV_CLOSED)) != 0) {
			event_warnx("%s: EV_SIGNAL is not compatible with "
			    "EV_READ, EV_WRITE or EV_CLOSED", __func__);
			return -1;
		}
		ev->ev_closure = EV_CLOSURE_EVENT_SIGNAL;
	} else {
		if (events & EV_PERSIST) {
			evutil_timerclear(&ev->ev_io_timeout);
			ev->ev_closure = EV_CLOSURE_EVENT_PERSIST;
		} else {
			ev->ev_closure = EV_CLOSURE_EVENT;
		}
	}

	min_heap_elem_init_(ev);

	if (base != NULL) {
		/* by default, we put new events into the middle priority */
		ev->ev_pri = base->nactivequeues / 2;
	}

	event_debug_note_setup_(ev);

	return 0;
}
```

5. 调用ops->enable，此处的ops->enable的作用是会将调用event_add函数将时间加入的事件队列中去
```
static int
event_listener_enable(struct evconnlistener *lev)
{
	struct evconnlistener_event *lev_e =
	    EVUTIL_UPCAST(lev, struct evconnlistener_event, base);
	return event_add(&lev_e->listener, NULL);
}
```

</font>

---

```dot
digraph G {
	node[shape=box]
	edge[]

	subgraph cluster_evconnlistener_new_bind{
		AStart[label=start]
		bind [color=red]
		AStart->createSocket->
		setSocketKeepAlive->
		bind->
		evconnlistener_new->Return
	}

	subgraph cluster_evconnlistener_new {
		callolc [label="new evconnlistener_event"]
		evconnlistener_new->BStart
		BStart[label=start]
		BStart->listen->callolc->init->event_assign->evconnlistener_enable
		evconnlistener_enable->evconnlistener_new [style=dashed,color=red]
	}

	descEvetnAssign [label="new a event, and init it with variables, like fd, events, callback functions",style=dashed,color=red,shape=box]
	event_assign->descEvetnAssign[style=dashed]

	descEvConnEnable[label="call event_listener_enable. In fact, it add listen event to table",style=dashed,color=red,shape=box]
	evconnlistener_enable->descEvConnEnable [style=dashed]

}
```
