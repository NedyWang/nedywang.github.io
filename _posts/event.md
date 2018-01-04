1.	event结构体
2.	event_add函数

========================

1. event结构体
<pre>struct event {
	struct event_callback ev_evcallback;
	/* for managing timeouts */
	union {
		TAILQ_ENTRY(event) ev_next_with_common_timeout;
		int min_heap_idx;
	} ev_timeout_pos;
	evutil_socket_t ev_fd;
	struct event_base *ev_base;
	union {
		/* used for io events */
		struct {
			LIST_ENTRY (event) ev_io_next;
			struct timeval ev_timeout;
		} ev_io;

		/* used by signal events */
		struct {
			LIST_ENTRY (event) ev_signal_next;
			short ev_ncalls;
			/* Allows deletes in callback */
			short *ev_pncalls;
		} ev_signal;
	} ev_;

	short ev_events;
	short ev_res;		/* result passed to event callback */
	struct timeval ev_timeout;
};
1. event_callback是1个结构体，定义了event的callback函数应该有的数据成员
2. ev_是1个联合体，可以是io事件，也可以是signal处理函数
3. ev_events，定义事件类型，是可读还是可写
4. ev_res，事件返回，已经就绪的事件
5. 时间相关....//TODO
</pre>
<pre>
struct event_callback {
	TAILQ_ENTRY(event_callback) evcb_active_next;
	short evcb_flags;
	ev_uint8_t evcb_pri;	/* smaller numbers are higher priority */
	ev_uint8_t evcb_closure;
	/* allows us to adopt for different types of events */
	union {
		void (*evcb_callback)(evutil_socket_t, short, void *);
		void (*evcb_selfcb)(struct event_callback *, void *);
		void (*evcb_evfinalize)(struct event *, void *);
		void (*evcb_cbfinalize)(struct event_callback *, void *);
	} evcb_cb_union;
	void *evcb_arg;
};
</pre>

2. event\_add实际是调用函数event_add_nolock_去执行工作的

<pre>
<code>
int
event_add_nolock_(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
</code>
1. add前准备
2. 调用函数add event
	1. 
3. add后处理
</pre>

