最近遇到一个问题，使用libssh2_sftp上传文件时，传输的文件命名文xxx.xx.temp，上传完成后需要把temp后缀去掉。
libssh2提供了一个接口`libssh2_sftp_rename`用于重命名远端文件。
但是在调用`libssh2_sftp_rename`时出现一个问题，只有第一次调用可以成功，后面在调用就会失败，返回错误码`-31`。
在`libssh2.h`里定义了该错误：
```cpp
#define LIBSSH2_ERROR_SFTP_PROTOCOL             -31
```
`libssh2_sftp_rename`的代码实现为：
```cpp
#define libssh2_sftp_rename(sftp, sourcefile, destfile) \
    libssh2_sftp_rename_ex((sftp), (sourcefile), strlen(sourcefile), \
                           (destfile), strlen(destfile),                \
                           LIBSSH2_SFTP_RENAME_OVERWRITE | \
                           LIBSSH2_SFTP_RENAME_ATOMIC | \
                           LIBSSH2_SFTP_RENAME_NATIVE)
```
而`libssh2_sftp_rename_ex`实现为：
```cpp
LIBSSH2_API int
libssh2_sftp_rename_ex(LIBSSH2_SFTP *sftp, const char *source_filename,
                       unsigned int source_filename_len,
                       const char *dest_filename,
                       unsigned int dest_filename_len, long flags)
{
    int rc;
    if(!sftp)
        return LIBSSH2_ERROR_BAD_USE;
    BLOCK_ADJUST(rc, sftp->channel->session,
                 sftp_rename(sftp, source_filename, source_filename_len,
                             dest_filename, dest_filename_len, flags));
    return rc;
}
```
`libssh2_sftp_rename`函数最终调用为`sftp_rename`，在`sftp_rename`里写明了返回`LIBSSH2_ERROR_SFTP_PROTOCOL`错误的几种情况：
+ 版本太低不支持
```cpp
if(sftp->version < 2) {
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "Server does not support RENAME");
    }
```

+ 重命名包长太小
```cpp
    rc = sftp_packet_require(sftp, SSH_FXP_STATUS,
                             sftp->rename_request_id, &data,
                             &data_len, 9);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    else if(rc == LIBSSH2_ERROR_BUFFER_TOO_SMALL) {
        if(data_len > 0) {
            LIBSSH2_FREE(session, data);
        }
        return _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                              "SFTP rename packet too short");
    }
```

+ 文件早已存在
```cpp
case LIBSSH2_FX_FILE_ALREADY_EXISTS:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "File already exists and "
                                 "SSH_FXP_RENAME_OVERWRITE not specified");
        break;
```

+ 操作不支持
```cpp
case LIBSSH2_FX_OP_UNSUPPORTED:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "Operation Not Supported");
        break;
```

+ SFTP 协议错误
```cpp
default:
        retcode = _libssh2_error(session, LIBSSH2_ERROR_SFTP_PROTOCOL,
                                 "SFTP Protocol Error");
        break;
```

从上面可以看到，如果重命名的文件已经存在的话，再想调用`libssh2_sftp_rename`会失败，这也是为什么只有第一次重命名会成功，在以后就报错。
因此解决方法是，在重命名之前检测远端是否存在重命名后的文件，如果存在的话删除掉已有的文件，或者重新命名一个新名字。