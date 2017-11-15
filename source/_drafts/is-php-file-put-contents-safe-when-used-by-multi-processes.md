title: PHP多进程中使用file_put_contents安全吗？
date: 2017-11-13 23:07:11
tags: [PHP]

---

# TL;DR

PHP多进程使用 `file_put_contents()` 方法记录日志时，使用追加模式（`FILE_APPEND`），简短的日志内容不会重叠，即能安全的记录日志内容。

如果愿意的话，也可以考虑在标记位中增加一个 `LOCK_EX`。

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

# 参考

[Is lock-free logging safe?](https://www.jstorimer.com/blogs/workingwithcode/7982047-is-lock-free-logging-safe)

[StackOverflow: Is file append atomic in UNIX?](https://stackoverflow.com/questions/1154446/is-file-append-atomic-in-unix)

[Are Files Appends Really Atomic?](http://www.notthewizard.com/2014/06/17/are-files-appends-really-atomic/)

[Appending to a File from Multiple Processes](http://nullprogram.com/blog/2016/08/03/)


