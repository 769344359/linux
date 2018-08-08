# 系统调用read
系统调用主要是
```
static ssize_t generic_file_buffered_read(struct kiocb *iocb,
		struct iov_iter *iter, ssize_t written)
{
...
page_cache_sync_readahead(mapping,
					ra, filp,
					index, last_index - index);
...
}
```
