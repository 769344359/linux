```
/* Process an incoming IP datagram fragment. */
int ip_defrag(struct net *net, struct sk_buff *skb, u32 user)
{
	struct net_device *dev = skb->dev ? : skb_dst(skb)->dev;
	int vif = l3mdev_master_ifindex_rcu(dev);
	struct ipq *qp;

	__IP_INC_STATS(net, IPSTATS_MIB_REASMREQDS);
	skb_orphan(skb);

	/* Lookup (or create) queue header */
	qp = ip_find(net, ip_hdr(skb), user, vif);
	if (qp) {
		int ret;

		spin_lock(&qp->q.lock);

		ret = ip_frag_queue(qp, skb);

		spin_unlock(&qp->q.lock);
		ipq_put(qp);
		return ret;
	}

	__IP_INC_STATS(net, IPSTATS_MIB_REASMFAILS);
	kfree_skb(skb);
	return -ENOMEM;
}
EXPORT_SYMBOL(ip_defrag);
```
`ip_defrag` 是一个将分片的数据组装成原来的数据.

主要委托给两个函数
- `ip_find` 类似于数据库操作的 createOrUpdate 函数, 如果有原来的包的队列就返回那个队列,否则则是生成一个队列
- `ip_frag_queue` 主要的处理都委托给 ip_frag_queue
***
## `ip_find` 的实现

在 `ip_find` 函数中, 根据 id  , sourceAddress , destinationAddress , protocol  生成hash值,然后委托给`inet_frag_find`
```
/* Find the correct entry in the "incomplete datagrams" queue for
 * this IP datagram, and create new one, if nothing is found.
 */
static struct ipq *ip_find(struct net *net, struct iphdr *iph,
			   u32 user, int vif)
{
	struct inet_frag_queue *q;
	struct ip4_create_arg arg;
	unsigned int hash;

	arg.iph = iph;
	arg.user = user;
	arg.vif = vif;
	 // 根据 id  , sourceAddress , destinationAddress , protocol  生成hash值
	hash = ipqhashfn(iph->id, iph->saddr, iph->daddr, iph->protocol);  

	q = inet_frag_find(&net->ipv4.frags, &ip4_frags, &arg, hash); // 委托给 inet_frag_find  函数
	if (IS_ERR_OR_NULL(q)) {
		inet_frag_maybe_warn_overflow(q, pr_fmt());
		return NULL;
	}
	return container_of(q, struct ipq, q);
}
```
其中 `inet_frag_queue` 实现是
```
struct inet_frag_queue *inet_frag_find(struct netns_frags *nf,
				       struct inet_frags *f, void *key,
				       unsigned int hash)
{
	struct inet_frag_bucket *hb;
	struct inet_frag_queue *q;
	int depth = 0;

	if (frag_mem_limit(nf) > nf->low_thresh)
		inet_frag_schedule_worker(f);

	hash &= (INETFRAGS_HASHSZ - 1);
	hb = &f->hash[hash];

	spin_lock(&hb->chain_lock);
	hlist_for_each_entry(q, &hb->chain, list) {
		if (q->net == nf && f->match(q, key)) {
			refcount_inc(&q->refcnt);
			spin_unlock(&hb->chain_lock);
			return q;
		}
		depth++;
	}
	spin_unlock(&hb->chain_lock);

	if (depth <= INETFRAGS_MAXDEPTH)
		return inet_frag_create(nf, f, key);

	if (inet_frag_may_rebuild(f)) {
		if (!f->rebuild)
			f->rebuild = true;
		inet_frag_schedule_worker(f);
	}

	return ERR_PTR(-ENOBUFS);
}
EXPORT_SYMBOL(inet_frag_find);
```

```
/**
 * hlist_for_each_entry	- iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry(pos, head, member)				\
	for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member);\
	     pos;							\
	     pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))
```
