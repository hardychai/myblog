在以前的文章里写了Windows下编译libcurl实现sftp传输文件，但是实际使用过程中遇到很大的问题，那就是传输速度非常慢，只有600KBps左右。然后用了Xftp和WinSCP两款软件做对比，Xftp勉强到1MBps，而WinSCP可以达到1.5MBps，传输速度都远大于我自己的传输速度。所以需要对它进行一下优化。

## 包大小与传输速度
在我的demo中，每次传包的大小为64KB。我尝试着把包大小增大到128KB直到512KB，速度都没有任何变化，都稳定维持在600KBps。
于是乎看了一下`libssh2_sftp_write`源码：
```cpp
LIBSSH2_API ssize_t
libssh2_sftp_write(LIBSSH2_SFTP_HANDLE *hnd, const char *buffer,
                   size_t count)
{
    ssize_t rc;
    if(!hnd)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, hnd->sftp->channel->session,
                 sftp_write(hnd, buffer, count));
    return rc;

}
```
`libssh2_sftp_write`源码调用的是`sftp_write`，在`sftp_write`里有这么一句话：
```cpp
uint32_t size = MIN(MAX_SFTP_OUTGOING_SIZE, count);
```
其中`count`是我们传入的大小，也就是64*1024，而`MAX_SFTP_OUTGOING_SIZE`宏定义是：
```cpp
/*
 * MAX_SFTP_OUTGOING_SIZE MUST not be larger than 32500 or so. This is the
 * amount of data sent in each FXP_WRITE packet
 */
#define MAX_SFTP_OUTGOING_SIZE 30000
```
也就是说最大传输长度才30000，不到32KB。而且根据注释，哪怕修改`MAX_SFTP_OUTGOING_SIZE`的数值，也不能超过32500。而32KB=32*1024=32768 > 32500，也就是说想要凑整把包大小设置为32KB也是不行的。

那就试试把包大小改小点。

当我把包大小改到25KB(25600)，好家伙，传输速度肉眼可见的提升，直接飙到了1.8MBps。我又尝试了几个不同的包大小设置，发现**24KB(24576)**时速度最快，达到了3MBps。

顺便提一下，在libcurl里设置上传包大小用的是`curl_easy_setopt(curl, CURLOPT_UPLOAD_BUFFERSIZE, buffer_size_long);`，而`buffer_size_long`大小范围固定为16KB\~2MB(curl\lib\setopt.c)，libssh2又规定了包大小不能大于32500，也就是说libcurl下传输包大小一般设置为16KB\~31KB。

所以最终把包大小设置为24KB，能满足速度要求。至于为什么，或者还有没有其他更好地包大小，以后再细细研究。