# share
rdma沉淀与积累
● 出发点
本文的目的是总结下RDMA编程的api以及个人在尝试时遇到的一些问题，主要参考的是rdma编程手册，rdma mojo和一些blog以及自己的理解。希望达到的效果是让一个不清晰rdma编程的人在看完后，能够写出个简易的rdma程序，同时也可以做一个查询手册的作用。
请求侧 = client 响应侧 = server
● 函数调用流程
连接管理使用rdma_cm的api，传输管理使用ib的api是一种常见的RDMA编程方式(实际上似乎IB已经是过时的头文件了，现在使用rdma_cm来处理所有操作，rdma_cm也有处理数据传输的能力，实际上就是对IB的一层封装)
如果此时我们想为两台机器建立rdma连接，该怎么做？
请求侧：首先请求侧先初始化addr里面放置了预设的端口和服务器的ip。
rdma_create_event_channel->rdma_create_id->rdma_resolve_addr(放入初始化的端口和ip)
之后rdma_get_cm_event, 处理事件RDMA_CM_EVENT_ADDR_RESOLVED
之后调用rdma_resolve_route->rdma_connect, 在connect之前通过会初始化好qp以及cq.
响应侧：首先得给一个端口号，在addr设置端口号。
rdma_bind_addr->rdma_listen 在接收到客户端的connect之后，处理事件RDMA_CM_EVENT_CONNECT_REQUEST，同样地配置好qp和cq之后，rdma_accpt建立连接。此时两侧都会产生
RDMA_CM_EVENT_ESTABLISHED事件。(unix socket编程是你listen接收到accpet连接就建立了，accpet只是取出一个连接)
如果此时我们想让客户端发送个简单的信息给服务器端，该怎么做？
响应侧：提前下发好receive queue element(rqe)，开启cq线程，令其阻塞在ibv_get_cq_event，记得之前得先调用ibv_req_notify_cq。
请求侧：malloc->填充内容->ibv_reg_mr->ibv_post_send, 之后send queue element(sqe)便会被rdma硬件拿出执行，并转化成completion queue element(cqe)放到cq中。
响应侧的rqe在接受到sqe后会被消耗，产生cqe，get_cq_event不再阻塞，得到响应事件IBV_WC_RECV，之后就可以从rqe的memory中拿到想要的内容了. 注意这里的post_send用的是send操作，事先不需要知道对端的地址空间。
● 连接管理
1 struct rdma_event_channel * rdma_create_event_channel (void)
注: 函数的目的是创建一个使能notify机制的通道。这个通道既可以用于rdma_get_cm_event，也可以用于ibv_get_cq_event。实际上这个channel的结构体里面有一个文件描述符，这个文件描述符与其他的无异。user可以通过poll来监听这个描述符，获取事件的输入。在性能方面，在大量的客户端场景下，还是建议使用多个通道对应多个cm_id以提高处理速度。通道创建后必须要手动销毁回收资源。
2 void rdma_destory_event_channel(struct rdma_event_channel *cc)  								
注：销毁事件通道。
3 int rdma_create_id(struct rdma_event_channel *channel, struct rdma_cm_id **id, void *context, enum rdma_port_space ps) 
注: 函数创建了一个cm_id，channel是之前创建的事件通知的通道。context是用户传进去的上下文，这是由于rdma的操作都是异步的，因此这个上下文在很多场景下非常有用。例如在获取到cm_event的时候，这个context可以通过返回的cm_id获取，可以得到更多的关于这个cm_id的上下文信息。rdma_port_space可以选TCP或者是UDP的，大致与实际的tcp和udp相同。在不使用rdma_cm的库时也可以用普通的socket建立连接。同样地也需要手动回收资源。
struct rdma_cm_id {
	struct ibv_context	*verbs; 
	struct rdma_event_channel *channel;
	void			*context;
	struct ibv_qp		*qp;
	struct rdma_route	 route;
	enum rdma_port_space	 ps;
	uint8_t			 port_num;
	struct rdma_cm_event	*event;
	struct ibv_comp_channel *send_cq_channel;
	struct ibv_cq		*send_cq;
	struct ibv_comp_channel *recv_cq_channel;
	struct ibv_cq		*recv_cq;
	struct ibv_srq		*srq;
	struct ibv_pd		*pd;
	enum ibv_qp_type	qp_type;
};
int rdma_destory_id(struct rdma_cm_id *id)
注：销毁cm_id。这里值得一提的是在销毁cm_id之前必须释放所有qp相关的资源以及ack相关的事件。通常是rdma_disconnect()释放连接，然后调用rdma_destory_qp，再调用rdma_destory_id。但如果在rdma_ack_cm_event之前调用这个函数，就会造成死锁。
int rdma_get_cm_event(struct rdma_event_channel *channel, struct rdma_cm_event **event)
注: 字面上就知道这个函数是来获取连接过程中的事件的。event的结构体如下.
struct rdma_cm_event {
	struct rdma_cm_id	*id;
	struct rdma_cm_id	*listen_id;
	enum rdma_cm_event_type	 event;
	int			 status;
	union {
		struct rdma_conn_param conn;
		struct rdma_ud_param   ud;
	} param;
};
其中id指的是实际连接的id，listen_id是监听的cm_id。event是事件类型，status是表示标识是否有错误发生，无误的话是0。param是一些额外信息，具体的可以看rdma编程手册，重点的是在于事件类型！
RDMA_CM_EVENT_ADDR_RESOLVED 
注：只会在发起请求那一端出现的事件。
      RDMA_CM_EVENT_ROUTE_RESOLVED
注: 只会在发求请求那一端出现的事件。									
      RDMA_CM_EVENT_CONNECT_REQUEST
注：只会在响应侧出现的事件，收到了连接请求。
RDMA_CM_EVENT_CONNECT_RESPONSE
注：只会在请求侧出现的事件，发送连接请求成功，通常不处理。
      RDMA_CM_EVENT_ESTABLISHED
注：建立连接时两侧都出现的事件，表示连接成功建立。																			
      RDMA_CM_EVENT_TIMEWAIT_EXIT
注：当你断开连接不释放cm_id资源就会出现这个事件。类似于tcp的timewait的作用，实际上qp在断开连接后会处于一个timewait的状态用来处理inflight包，timewait结束就会触发该事件。如果你直接销毁cm_id就不会有这个事件了.
      RDMA_CM_EVENT_DISCONNECTED
注：断开连接时两侧都出现的事件。
      RDMA_CM_EVENT_UNREACHABLE
注：服务器端没启动就开启客户端就会出现这个事件。
RDMA_CM_EVENT_REJECTED
      注：字面意思是请求被远端设备拒绝。这个事件可能是由于客户端访问了错误的端口，只会在请求侧出现。
																					
4 int rdma_ack_cm_event(struct rdma_cm_event *event)
注：与rdma_get_cm_event配套，该操作会销毁回收event的所有资源。因此在该函数后不得再去访问event,有可能会超出函数调用栈造成段错误！
5  rdma_bind_addr(struct rdma_cm_id *id, struct sockaddr *addr )
注:字面意思绑定local地址。不绑定的话就是一个通配符，addr里至少得指定一个端口（可以不指定ip）。如果端口是0的话则由os随机选择个空闲端口。服务器端在rdma_listen前调用该函数。
6 int rdma_resolve_addr (struct rdma_cm_id *id, struct sockaddr *src_addr, struct sockaddr *dst_addr, int timeout_ms) 
注: 这个函数的作用是将目的地址映射成rdma地址，位于请求侧。src_addr可以不绑定，设置成NULL，如要绑定需调用rdma_bind_addr.
7 int rdma_resolve_route (struct rdma_cm_id *id, int timeout_ms) 
注: 需调用在reslove_addr之后，位于请求侧。
8 int rdma_listen(struct rdma_cm_id *id, int backlog) 
注：在调用listen之前必须绑定地址，如指定了ip地址则只会监听对应的设备，如未指定则监听所有。值得一提的是在调用了listen后cm_id中的verbs也就是操作rdma设备的上下文的值会发生改变，但需要指定ip即指定哪个设备，否则只会在接受了连接请求后才会对verbs赋值。实际上这个context与设备相关，ibv_get_device获取到rdma设备指针后，通过ibv_open_device就可以得到context，同一个设备返回的是同一个context。换句话说设备与上下文一一对应。
9 int rdma_accept(struct rdma_cm_id *id, struct rdma_conn_param *conn_param)
注: accept调用在cm event connection request之后，需要强调的是listen_cm id在接受到新的连接请求后，event结构体中的id是一个新的cm_id。rdma_accept的是这个新的cm_id，这个新的cm_id已绑定在某个rdma设备上，因此有ibv context。
10 int rdma_connect(struct rdma_cm_id *id, struct rdma_conn_param *conn_param) 
注: 这个函数是请求侧请求连接，需要调用在resolve_route之后，没有特别需要注意的地方。可以在conn_param中指定qp_num和srq. 但意义不大，因为不是在服务器侧。
11  int rdma_disconnect(struct rdma_cm_id *id)  
注：断开连接，两边都会产生RDMA_CM_EVENT_DISCONNECTED。执行完这个函数后会将与之关联的qp的状态改成error，同时会将所有请求变成cqe放入cq中。
12 int rdma_create_qp (struct rdma_cm_id *id, struct ibv_pd *pd, struct ibv_qp_init_attr *qp_init_attr)   void rdma_destroy_qp (struct rdma_cm_id *id) 
注: 主要需要注意的点就是attr的qp_type和sq_sig_all. qp_type分为三种1. RC 2. UD 3. UC。RC类似于TCP保证了可靠性传输，通常用于文件系统。UD和UC都是不可靠连接，UD类似于udp可以用于组播。sq_sig_all只有两种值有效，0和非0，如果非0则所有send工作请求都会产生cqe，是0的话则只有wr的send_flags设置成IBV_SEND_SIGNALED的才会产生cqe。srq是共享接受队列，因为通常接受队列是比较少用的（出于效率考虑，一般传输用的单端操作即write和read）,为了接受资源设计了srq使得多个qp可以共享一个rq.
struct ibv_qp_init_attr {
	void		       *qp_context;
	struct ibv_cq	       *send_cq;
	struct ibv_cq	       *recv_cq;
	struct ibv_srq	       *srq;
	struct ibv_qp_cap	cap;
	enum ibv_qp_type	qp_type;
	int			sq_sig_all;
};
struct ibv_qp_cap {
	uint32_t		max_send_wr;
	uint32_t		max_recv_wr;
	uint32_t		max_send_sge;
	uint32_t		max_recv_sge;
	uint32_t		max_inline_data;
};
send queue 和 receive queue的最大长度是与硬件相关的，可以通过ibv_devinfo -v来获取硬件的相关信息，但实际上真正可用的大小往往比这个最大长度要小，唯一的方法就是一个个试直至不报错。max_send_sge和max_recv_sge根据程序需要设置，也是跟硬件相关的。至于max_inline_data，他指的是内联数据。内联数据不需要注册内存就可直接发送，它的最大值只能通过试错法去寻找。
返回的qp指针中有一个qp_num域，它是个24位的整形，标识着一个qp，可以用来在基于poll的单cq单cc的rdma服务器用来找到返回cqe的是哪一个qp，不足的是用户无法去修改这个值。
创建qp时类型的不同也会影响到max_send_wr和max_recv_wr，只能用试错法一个个找。
13 char *rdma_event_str (enum rdma_cm_event_type event) 
注：这个是用来将枚举变量做一个逆翻译告知具体错误是什么的。
● 传输管理
1  struct ibv_pd *ibv_alloc_pd(struct ibv_context *context)  
注：创建一个内存保护域PD，这在之后的ibv_reg_mem中会用到。这里唯一需要提的就是如果发生段错误，多半是context还是0.还有就是在连接中断后记得destory pd。
2 struct ibv_comp_channel *ibv_create_comp_channel(struct ibv_context *context) 
注: 创建一个cq的事件通道，简写成cc。get_cq_event监听的是cc，因此可以将相同优先级或者是相同工作内容的cq放在一个cc里面。多线程场景下如果多cq共享一个cc，那么哪一个线程get the event是未知的 。
3 struct ibv_cq *ibv_create_cq(struct ibv_context *context, int cqe, void *cq_context, struct ibv_-  comp_channel *channel, int comp_vector) 
注：创建一个cq，这里指的提的一点是一个cc可以被多个cq所共享，cqe指的是cq至少的长度，是一个下界，取值在[1,max_cqe]之间，max_cqe是硬件固有的值，这里其实很奇怪，可以简单把它就当作cq的大小，但实际上rdma会分配更多一点内存，也就是会比这个预先设置队列大小要大。当cqe满时会产生IBV_EVENT_CQ_ERR事件。cq可以被多个sq和rq共享，通常出于减少上下文切换的考虑会共享一个cq。
4 struct ibv_srq *ibv_create_srq(struct ibv_pd *pd, struct ibv_srq_init_attr *srq_init_attr) 
注：创建一个srq，这个是用了共享接受队列。srq有自己特有的post_srq_recv函数。他是被单独创建，然后让其他qp的srq指针指向它。创建它时，需要预设置它的一些属性。
struct ibv_srq_attr {
        uint32_t                max_wr;
        uint32_t                max_sge;
        uint32_t                srq_limit; //不用管 仅仅是iwarp相关的
};
5 int ibv_post_recv(struct ibv_qp *qp, struct ibv_recv_wr *wr, struct ibv_recv_wr **bad_wr) 
注：该函数的作用是发送一个rqe到receive queue。在调用该函数前，必须要先初始化好ibv_recv_wr。
struct ibv_recv_wr {
	uint64_t		wr_id;
	struct ibv_recv_wr     *next;
	struct ibv_sge	       *sg_list;
	int			num_sge;
};
wr_id是用来标识一个rqe的整型，一般不用设置。*next指代下一个rqe。sg_list实际上是一个ibv_sge的数组。num_sge指代使用该数组中多少个元素（这里的元素实际上就是地址空间）.

上图是send_wr的模型，但recv_wr也是一样的。从图上可以看出一个wr可以对应多个sge(内存块).ibv_sge是一个结构体，它里面的内容是注册好的内存起始地址，长度，以及lkey(由mr返回得到). RDMA硬件会自动将这些离散的内存块聚合成连续的，可能post_send或者recv有多个wr，但这些wr都是独立的。
struct ibv_sge {
        uint64_t        addr;
        uint32_t        length;
        uint32_t        lkey;
};
rqe只有在三种情况下会被转变成cqe，1. Send  2. Send with Immediate 3. RDMA Write with immediate。注意如果sge中有未注册的内存会立即产生cqe，cqe中标记错误号为IBV_WC_LOC_PROT_ERR.如果在调用时发生了段错误，多半是sge和wr的next是无效的地址。
7 int ibv_post_send(struct ibv_qp *qp, struct ibv_send_wr *wr, struct ibv_send_wr **bad_wr) 
注：该函数的作用是发送一个sqe到send queue。和ibv_recv_wr一样要先初始化ibv_send_wr。
struct ibv_send_wr {
        uint64_t                wr_id;                  /* User defined WR ID */
        struct ibv_send_wr     *next;                   /* Pointer to next WR in list, NULL if last WR */
        struct ibv_sge         *sg_list;                /* Pointer to the s/g array */
        int                     num_sge;                /* Size of the s/g array */
        enum ibv_wr_opcode      opcode;                 /* Operation type */
        int                     send_flags;             /* Flags of the WR properties */
        uint32_t                imm_data;               /* Immediate data (in network byte order) */
        union {
                struct {
                        uint64_t        remote_addr;    /* Start address of remote memory buffer */
                        uint32_t        rkey;           /* Key of the remote Memory Region */
                } rdma;
                struct {
                        uint64_t        remote_addr;    /* Start address of remote memory buffer */
                        uint64_t        compare_add;    /* Compare operand */
                        uint64_t        swap;           /* Swap operand */
                        uint32_t        rkey;           /* Key of the remote Memory Region */
                } atomic;
                struct {
                        struct ibv_ah  *ah;             /* Address handle (AH) for the remote node address */
                        uint32_t        remote_qpn;     /* QP number of the destination QP */
                        uint32_t        remote_qkey;    /* Q_Key number of the destination QP */
                } ud;
        } wr;
};
send_flags
      IBV_SEND_SIGNALED：仅在qp的sq_sig_all设置成0有效，如果sq_sig_all为0，那么只有send_flags设置成SIGNALED时才会产生cqe，注意这是在本地返回的cqe，需要注意的是这样操作虽然不会产生cqe，但是也不会消耗sqe
signaled是优化local的。
IBV_SEND_SOLICITED： 用于减少cpu开销的，如果你在send_flags设置了该位，仅当remote的req_notify_cq的第二个参数非0才有意义。那么get_cq_event仅在两种情况下不再阻塞，一个是接受到含有该标记位置的sq，一种是本地的wqe在放入cq中时有错误。solicited是优化remote的。

IBV_SEND_INLINE：sg_list 中指定的内存缓冲区将内联数据放置在发送请求中。 这意味着是用低级驱动程序（即 CPU）将读取数据而不是 RDMA 设备。 这意味着不会检查 L_Key，实际上这些内存缓冲区甚至不必注册，它们可以在 ibv_post_send() 结束后立即重用，仅对send和write有效。
opcode
IBV_WR_SEND
双端操作，指的是不仅会在local消耗sqe产生cqe，也会在remote消耗rqe产生cqe。send就是一种典型的双端操作，send方并不需要知道对端用来接受的内存地址和长度，它只会单纯的消耗掉对端receive queue头部的rqe。如果对端的rqe的内存链的容量不足，则会产生error。对于RC和UC来说，一次发送的message的大小在[0,2^31], 而对于UD则是取决于链路层的mtu。
IBV_WR_SEND_WITH_IMM
类似于IBV_WR_SEND，不同的是它会将imm_data字段的值发到对端。这里值得一提的是发到对端后触发的依然是IBV_WC_RECV而不是IBV_WC_RECV_IMM。
IBV_WR_RDMA_WRITE
单端操作，不会消耗remote的rqe。write需要事先知道remote的起始地址和rkey，注意指的地址全都是逻辑地址，也就是说在物理内存上这一块不一定是连续的。与send一样最大的message大小[0, 2^31]
IBV_WR_RDMA_WRITE_WITH_IMM
双端操作，类似于IBV_WR_RDMA_WRITE，但会消耗对端的rqe产生cqe事件。但与Send_with_IMM不同的是他不会使用receive_wr中的sge。
IBV_WR_RDMA_READ
和WRITE一样是单端操作，读取远程一块连续内存的内容，与write一样需要事先知道remote的起始地址和rkey。
8 int ibv_req_notify_cq(struct ibv_cq *cq, int solicited_only) 
注：这个函数是将cq的状态设置为notify，如果不改变cq的状态那么ibv_get_cq_event将不会感知到任何cqe,即使有新的cqe被放到cq中。当get_event之后，会自动将cq的状态reset，因此需要再度调用该函数用于下一次事件到达。因为它的本质是改变cq的状态，因此你多次调用请求相同的事件类型，那么实际的效果是一样的（最多支持两种不同的事件类型，指的是solicited与非solicited，不同的事件类型是可以分开的）。注意该函数仅仅是与rqe相关联，也就是说它请求的事件是send，write with imm_data等一些会消耗rqe的wr。如果在调用该函数前已经有cqe在cq中，且没有新的cqe到达，那么get_cq_event是无法感知到有事件完成了的。
9 int ibv_get_cq_event(struct ibv_comp_channel *channel, struct ibv_cq **cq, void **cq_con- 
text) 
注: 这个函数默认情况下是阻塞的，直到有一个cqe被放入到了cq中（也可以通过fcntl将其设置成非阻塞，并结合select/poll/epoll)。如果想要得到一个cq事件，你必须在此之前主动调用req_notify_cq去请求一个cq事件。一旦接收到一个事件通知那么cq就会变成disarm状态，需要再度req_notify_cq，将其状态转变成rearmed状态用于接受下一次事件通知。
10 int ibv_poll_cq(struct ibv_cq *cq, int num_entries, struct ibv_wc *wc) 
注：在调用cq_event后，实际是并没有正式处理cqe，它所返回的仅仅是哪一个cq有事件发生。通常是调用ibv_poll_cq来进行cqe的处理。
struct ibv_wc {
	uint64_t		wr_id;
	enum ibv_wc_status	status;
	enum ibv_wc_opcode	opcode;
	uint32_t		vendor_err;
	uint32_t		byte_len;
	uint32_t		imm_data;
	uint32_t		qp_num;
	uint32_t		src_qp;
	int			wc_flags;
	uint16_t		pkey_index;
	uint16_t		slid;
	uint8_t			sl;
	uint8_t			dlid_path_bits;
};
通常用一个while循环来polling cq中的cqe。其结构体如上，byte_len指的是接收到的字节数，imm_data是sqe的立即数。qp_num是local的qp号，src_qp是远端的qp号。上述这些域仅在状态成功时全部有效，如果是错误的状态那么只有wr_id, status, qp_num,vendor_err有效。
status
IBV_WC_SUCCESS (0) 成功
IBV_WC_LOC_PROT_ERR (4) 本地内存访问到了未注册的内存。
IBV_WC_LOC_ACCESS_ERR (8)  本地内存权限不对
IBV_WC_REM_ACCESS_ERR (10) 远端内存权限不对
opcode
IBV_WC_SEND local的send的sqe转变成cqe。
IBV_WC_RDMA_WRITE local的write的sqe转变成cqe。
IBV_WC_RDMA_READ local的read的sqe转变成cqe。
IBV_WC_RECV 用于接收ibv_send 和 ibv_send_with_imm。
IBV_WC_RECV_RDMA_WITH_IMM 用于接收ibv_write_with_imm。
12 void ibv_ack_cq_events(struct ibv_cq *cq, unsigned int nevents) 
注：必须对每个get_cq_event获取到的事件进行确认，若不确认，在销毁cq时会陷入阻塞。由于ack事件需要用到互斥锁，因此时间开销是非常大的。
13 struct ibv_mr *ibv_reg_mr(struct ibv_pd *pd, void *addr, size_t length, enum ibv_access_flags access) 
注：就是注册内存，rdma中无论是用来发送还是接受的内存区域都必须要通过注册受到保护，防止os对这一块区域换页，或者是修改。可以多次注册同一个内存块或其中的一部分。 甚至可以使用不同的访问标志来执行这些内存注册。
IBV_ACCESS_LOCAL_WRITE 允许本地写操作
IBV_ACCESS_REMOTE_WRITE 允许remote写操作，当enable该选项时，LOCAL_WRITE也必须是enable的。
IBV_ACCESS_REMOTE_READ  允许remote读操作。
LOCAL_READ 永远是被允许的。    
14 int ibv_destroy_comp_channel(struct ibv_comp_channel *channel) 
注：在断开连接之后,销毁comp_channel.
15 int ibv_destroy_cq(struct ibv_cq *cq) 
注：在断开连接之后,销毁cq.
。。。 还有一些销毁内存的操作就不复述了，总之在断开连接后需要回收cc，cq，qp，pd，mr等资源

● 几种操作
  Send 双端操作
    搬运下好图
  
1. 接收端APP以WQE的形式下发一次接收任务。
2. 接收端硬件从RQ中拿到任务书，准备接收数据。
3. 发送端APP以WQE的形式下发一次SEND任务。
4. 发送端硬件从SQ中拿到任务书，从内存中拿到待发送数据，组装数据包。
5. 发送端网卡将数据包通过物理链路发送给接收端网卡。
6. 接收端收到数据，进行校验后回复ACK报文给发送端。
7. 接收端硬件将数据放到WQE中指定的位置，然后生成“任务报告”CQE，放置到CQ中。
8. 接收端APP取得任务完成信息。
9. 发送端网卡收到ACK后，生成CQE，放置到CQ中。
10. 发送端APP取得任务完成信息。
    双端操作在没有流控的条件下可能会产生rnr流，即接受端还没下发rqe流已经到达了对端。
  Recieve 双端操作与Send类似
  Write 单端操作  

● 请求端APP以WQE（WR）的形式下发一次WRITE任务。
● 请求端硬件从SQ中取出WQE，解析信息。
● 请求端网卡根据WQE中的虚拟地址，转换得到物理地址，然后从内存中拿到待发送数据，组装数据包。
● 请求端网卡将数据包通过物理链路发送给响应端网卡。
● 响应端收到数据包，解析目的虚拟地址，转换成本地物理地址，解析数据，将数据放置到指定内存区域。
● 响应端回复ACK报文给请求端。
● 请求端网卡收到ACK后，生成CQE，放置到CQ中。
● 请求端APP取得任务完成信息
    注意这里的ack指的是网卡底层传输自动发的ack，不是那个ack_event，我们上层看的是message的传输实际上底层的依然是分包进行传输和确认的，rdma硬件有自己的一套协议栈。
  Read 单端操作与Write类似
● 性能相关思考
Solicited Events
使用该标志位可以让对端减少事件通知，但我没有使用过。实际上更多的还是用write和read这种不会消耗rqe的操作码来进行大批量的数据发送。
INLINE DATA
使用内联数据可以使得我们不用注册内存，或许能够降低空间开销。
IBV_SEND_SIGNALED 
虽然能够不触发本地的get_cq_event，但是却不消耗sqe，这导致了当sq满时，你再调用post_send会直接产生错误。
REF：
RDMA MOJO
RDMA编程手册
https://zhuanlan.zhihu.com/p/141267386
https://zhuanlan.zhihu.com/p/55142568
