title: PHP多进程中使用file_put_contents安全吗？
date: 2017-11-13 23:07:11
tags: [PHP]

---

# TL;DR

Linux下，PHP多进程使用 `file_put_contents()` 方法记录日志时，使用追加模式（`FILE_APPEND`），简短的日志内容不会重叠，即能安全的记录日志内容。

`file_put_contents()` 使用 `write()` 系统调用实现数据的写入，`write()` 系统调用对普通文件保证写入数据的完整性，`O_APPEND` 打开模式保证数据写入到文件末尾。

如果愿意的话，也可以考虑在标记位中使用 `LOCK_EX`。

<!-- more -->

# 从monolog说起

提起 PHP 日志记录，不得不说到 [monolog](https://github.com/Seldaek/monolog/) 这个项目，这几乎是现有大多数项目首选的日志库。

对于日志记录这一场景，无论是 HTTP API 还是 daemon 进程，在应用中总会遇到多个进程的情况。

PHP-FPM 下会存在多个 worker，而 daemon 常选择使用多进程的方式充分利用资源。多个进程之间的竞争是必然存在的，而 monolog 是如何解决的呢？

答案是文件锁。

翻看源码 [StreamHandler.php](https://github.com/Seldaek/monolog/blob/master/src/Monolog/Handler/StreamHandler.php)：

```
protected function write(array $record)
{
    if (!is_resource($this->stream)) {
        if (null === $this->url || '' === $this->url) {
            throw new \LogicException('Missing stream url, the stream can not be opened. This may be caused by a premature call to close().');
        }
        $this->createDir();
        $this->errorMessage = null;
        set_error_handler([$this, 'customErrorHandler']);
        $this->stream = fopen($this->url, 'a');
        if ($this->filePermission !== null) {
            @chmod($this->url, $this->filePermission);
        }
        restore_error_handler();
        if (!is_resource($this->stream)) {
            $this->stream = null;
            throw new \UnexpectedValueException(sprintf('The stream or file "%s" could not be opened: '.$this->errorMessage, $this->url));
        }
    }
    if ($this->useLocking) {
        // ignoring errors here, there is not much we can do about them
        // 注意，此处使用了阻塞的排他文件锁，多进程时等待
        flock($this->stream, LOCK_EX);
    }
    // 常规的文件写入操作
    $this->streamWrite($this->stream, $record);
    if ($this->useLocking) {
        // 写入完成后解锁
        flock($this->stream, LOCK_UN);
    }
}

protected function streamWrite($stream, array $record)
{
    fwrite($stream, (string) $record['formatted']);
}
```

文件通过 `a` 模式，即追加模式打开，写入操作使用的是常规的 fwrite 操作。

让人困惑的是，已经使用 `a` 模式打开为何还需要上锁？这一个上锁操作来源于 GitHub 上的这一个 [issue #379](https://github.com/Seldaek/monolog/issues/379)。

`#379` 这个 issue 简而言之即用户在使用过程中发现写入一定长度的日志时出现了重叠的情况，于是提交了一个需要上锁的 PR。但是个人认为此处需要上锁的理由并不充分，因为 issue 中提到的问题，个人理解并不能确定是否是因为未上锁引起的。

有人说如果进程写日志过程中挂了没有解锁怎么办？没关系，文件锁在进程退出之后就会被释放。

## file_put_contents()的实现

### file_put_contents()完成的是open/write/close

翻阅 PHP `5.4.41` 源码中的 `ext/standard/file.c` 文件，可以看到 file_put_contents() 的实现(源码稍长，只做部分摘录)：

```
PHP_FUNCTION(file_put_contents)
{
	php_stream *stream; // 流的结构体，告知了流的读写操作参数
	// ...
	char mode[3] = "wb"; // 打开流的标记，默认是写二进制文件格式

	// ...

	context = php_stream_context_from_zval(zcontext, flags & PHP_FILE_NO_DEFAULT_CONTEXT);
   // 如果提供了 FILE_APPEND 标记为则以追加模式打开流
	if (flags & PHP_FILE_APPEND) {
		mode[0] = 'a';
	} else if (flags & LOCK_EX) { // 如果有 LOCK_EX标志则尝试对流上锁
		/* check to make sure we are dealing with a regular file */
		// ...
		mode[0] = 'c';
	}
	mode[2] = '\0';

	stream = php_stream_open_wrapper_ex(filename, mode, ((flags & PHP_FILE_USE_INCLUDE_PATH) ? USE_PATH : 0) | REPORT_ERRORS, NULL, context);
	
	// ...
	switch (Z_TYPE_P(data)) {
		case IS_RESOURCE: {
			// ...
			break;
		}
		case IS_NULL:
		case IS_LONG:
		case IS_DOUBLE:
		case IS_BOOL:
		case IS_CONSTANT:
			convert_to_string_ex(&data);

		case IS_STRING:
			if (Z_STRLEN_P(data)) {
				// 关键逻辑，实际写入操作
				numbytes = php_stream_write(stream, Z_STRVAL_P(data), Z_STRLEN_P(data));
				if (numbytes != Z_STRLEN_P(data)) {
					php_error_docref(NULL TSRMLS_CC, E_WARNING, "Only %ld of %d bytes written, possibly out of free disk space", numbytes, Z_STRLEN_P(data));
					numbytes = -1;
				}
			}
			break;

		case IS_ARRAY:
			// ...
			break;

		case IS_OBJECT:
			// ...
		default:
			numbytes = -1;
			break;
	}
	php_stream_close(stream);

	if (numbytes < 0) {
		RETURN_FALSE;
	}

	RETURN_LONG(numbytes);
}
```

可以看出，`file_put_contents()` 实际上是完成了 open -> write -> close 三大操作。

### 写入操作的实现

我们最为关心的 write 操作，跟踪源码可以发现，实际上是流结构体中的 write 函数指针指向的函数完成的：

```
static size_t _php_stream_write_buffer(php_stream *stream, const char *buf, size_t count TSRMLS_DC)
{
	size_t didwrite = 0, towrite, justwrote;

	// ...

	while (count > 0) {
		towrite = count;
		if (towrite > stream->chunk_size)
			towrite = stream->chunk_size;

		// 请注意此处
		justwrote = stream->ops->write(stream, buf, towrite TSRMLS_CC);
		// ...
```

那么问题来了，write 指向的函数到底是什么呢？

继续跟踪源码，在函数 `_php_stream_open_wrapper_ex()` 中找到了一些线索：

```
PHPAPI php_stream *_php_stream_open_wrapper_ex(char *path, char *mode, int options,
		char **opened_path, php_stream_context *context STREAMS_DC TSRMLS_DC)
{
	php_stream *stream = NULL;
	php_stream_wrapper *wrapper = NULL;
	// ...
	// 生成 wrapper 结构体
	wrapper = php_stream_locate_url_wrapper(path, &path_to_open, options TSRMLS_CC);
	// ...

	if (wrapper) {
		if (!wrapper->wops->stream_opener) {
			php_stream_wrapper_log_error(wrapper, options ^ REPORT_ERRORS TSRMLS_CC,
					"wrapper does not support stream open");
		} else {
			// 通过结构体中的 wops 中的 stream_opener 指向的函数完成流结构体的生成工作
			stream = wrapper->wops->stream_opener(wrapper,
				path_to_open, mode, options ^ REPORT_ERRORS,
				opened_path, context STREAMS_REL_CC TSRMLS_CC);
		}
	// ...
```

在 `main/stream/stream.c` 文件中的 `php_stream_locate_url_wrapper()` 函数中可以看到，对于文件，实际上返回的的是 `php_plain_files_wrapper` 的全局变量的指针：

```
PHPAPI php_stream_wrapper *php_stream_locate_url_wrapper(const char *path, char **path_for_open, int options TSRMLS_DC)
{
	// ...
	if (!protocol || !strncasecmp(protocol, "file", n))	{
		/* fall back on regular file access */
		php_stream_wrapper *plain_files_wrapper = &php_plain_files_wrapper;

		// ...

		return plain_files_wrapper;
	}
```

而这个变量的结构实际上包含了一个静态变量 `php_plain_files_wrapper_ops`：

```
static php_stream_wrapper_ops php_plain_files_wrapper_ops = {
	php_plain_files_stream_opener,
	NULL,
	NULL,
	php_plain_files_url_stater,
	php_plain_files_dir_opener,
	"plainfile",
	php_plain_files_unlink,
	php_plain_files_rename,
	php_plain_files_mkdir,
	php_plain_files_rmdir,
	php_plain_files_metadata
};

php_stream_wrapper php_plain_files_wrapper = {
	&php_plain_files_wrapper_ops,
	NULL,
	0
};
```

当中的 `php_plain_files_stream_opener` 函数指针指向的函数则明确的告知了如何生成流对象的实现：

```
static php_stream *php_plain_files_stream_opener(php_stream_wrapper *wrapper, char *path, char *mode,
		int options, char **opened_path, php_stream_context *context STREAMS_DC TSRMLS_DC)
{
	if (((options & STREAM_DISABLE_OPEN_BASEDIR) == 0) && php_check_open_basedir(path TSRMLS_CC)) {
		return NULL;
	}

	return php_stream_fopen_rel(path, mode, opened_path, options);
}
```

在流打开的函数 `_php_stream_fopen()` 中（位于文件 `main/stream/plain_wrapper.c`中），我们终于找到了生成流结构的逻辑：

```
PHPAPI php_stream *_php_stream_fopen(const char *filename, const char *mode, char **opened_path, int options STREAMS_DC TSRMLS_DC)
{
	// ...

	fd = open(realpath, open_flags, 0666);

	if (fd != -1)	{

		if (options & STREAM_OPEN_FOR_INCLUDE) {
			// 最终都会调用这一函数
			ret = php_stream_fopen_from_fd_int_rel(fd, mode, persistent_id);
		} else {
			// 注意此处，ret即生成的流结构，即最初实现方法中的stream变量的值
			ret = php_stream_fopen_from_fd_rel(fd, mode, persistent_id);
		}
		
		if (ret)	{
			// ...
			return ret;
		}
		close(fd);
	}
	efree(realpath);
	if (persistent_id) {
		efree(persistent_id);
	}
	return NULL;
}
```

再深入一步，看看 `_php_stream_fopen_from_fd_int()` (最终都会调用这一函数)这些函数是如何生成流结构中的 ops 结构体的：

```
static php_stream *_php_stream_fopen_from_fd_int(int fd, const char *mode, const char *persistent_id STREAMS_DC TSRMLS_DC)
{
	php_stdio_stream_data *self;

	self = pemalloc_rel_orig(sizeof(*self), persistent_id);
	memset(self, 0, sizeof(*self));
	self->file = NULL;
	self->is_pipe = 0;
	self->lock_flag = LOCK_UN;
	self->is_process_pipe = 0;
	self->temp_file_name = NULL;
	self->fd = fd;

   // php_stream_stdio_ops 就是我们想要找到ops操作体
	return php_stream_alloc_rel(&php_stream_stdio_ops, self, persistent_id, mode);
}

PHPAPI php_stream *_php_stream_alloc(php_stream_ops *ops, void *abstract, const char *persistent_id, const char *mode STREAMS_DC TSRMLS_DC) /* {{{ */
{
	php_stream *ret;

	ret = (php_stream*) pemalloc_rel_orig(sizeof(php_stream), persistent_id ? 1 : 0);

	memset(ret, 0, sizeof(php_stream));

	ret->readfilters.stream = ret;
	ret->writefilters.stream = ret;

	// ...

	// ops即 _php_stream_fopen_from_fd_int 传入的 php_stream_stdio_ops
	ret->ops = ops;
	// ...
	return ret;
}
```

write 操作的实现的答案就在 `php_stream_stdio_ops` 这一变量中:

```
PHPAPI php_stream_ops	php_stream_stdio_ops = {
	php_stdiop_write, php_stdiop_read,
	php_stdiop_close, php_stdiop_flush,
	"STDIO",
	php_stdiop_seek,
	php_stdiop_cast,
	php_stdiop_stat,
	php_stdiop_set_option
};
```

`php_stdiop_write` 函数指针指向的函数就是我们要的答案：
	
```
static size_t php_stdiop_write(php_stream *stream, const char *buf, size_t count TSRMLS_DC)
{
	php_stdio_stream_data *data = (php_stdio_stream_data*)stream->abstract;

	assert(data != NULL);

	if (data->fd >= 0) {
		// 最终调用 write 系统调用
		int bytes_written = write(data->fd, buf, count);
		if (bytes_written < 0) return 0;
		return (size_t) bytes_written;
	} else {

#if HAVE_FLUSHIO
		if (!data->is_pipe && data->last_op == 'r') {
			fseek(data->file, 0, SEEK_CUR);
		}
		data->last_op = 'w';
#endif

		return fwrite(buf, 1, count, data->file);
	}
}
```

跟踪到这里，得到了最终的结论:

**file_put_contents() 使用 write() 系统调用实现了数据的写入。**

## 写入安全的保证

造成多进程写入文件内容错误乱的原因很大程度上是因为每个进程打开文件描述符对应的文件位置指针都是独立的，如果没有同步机制，可能后来的写入的位置就会覆盖之前写入的数据，那么 `write()` 和 `O_APPEND` 能不能解决这个问题呢？

《Linux系统编程》第二章提到：

> 对于普通文件，除非发生一个错误，否则write()将保证写入所有的请求。
> ...
> 当fd在追加模式下打开时（通过指定O_APPEND参数），写操作就不从文件描述符的当前位置开始，而是从当前文件末尾开始。
> ...
> 它保证文件位置总是指向文件末尾，这样所有的写操作总是追加的，即便有多个写者。你可以认为每个写请求之前的文件位置更新操作是原子操作。

以上说明了：

+ 每个写操作由操作系统保证完成性，即进程 A 写入 aa，进程 B 写入 bb，文件中不可能出现类似的 `abab` 这样的数据交叉情况。
+ O_APPEND在多个写入者的情况下已然能保证数据写入文件末尾。

# 结论

综上，可以放心的使用 PHP 的 `file_put_contents()` 结合 `FILE_APPEND` 记录日志。

当然这是对于写入普通文件，如果写入的是管道则要关注是否数据大小超过 `PIPE_BUF` 的值了，这里有一篇有趣的博文 [Are Files Appends Really Atomic?](http://www.notthewizard.com/2014/06/17/are-files-appends-really-atomic/) 可以读读。

# 参考

[《PHP扩展开发与内核应用》：15 流的实现](http://www.cunmou.com/phpbook/15.md)

[《Linux 系统编程》（第二版）](https://book.douban.com/subject/25828773/)

[Is lock-free logging safe?](https://www.jstorimer.com/blogs/workingwithcode/7982047-is-lock-free-logging-safe)

[StackOverflow: Is file append atomic in UNIX?](https://stackoverflow.com/questions/1154446/is-file-append-atomic-in-unix)

[Appending to a File from Multiple Processes](http://nullprogram.com/blog/2016/08/03/)


