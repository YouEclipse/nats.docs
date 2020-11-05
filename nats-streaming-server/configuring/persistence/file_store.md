# 文件存储

为了跟高级别的投递，消息服务器应该配置文件存储。NATS Streaming提供了一个基础的文件存储的实现，未来将添加不同的文件存储的实现。

让NATS Streaming server使用文件存储持久化，需要在服务启动的时候提供两个参数:

```bash
nats-streaming-server -store file -dir datastore
```

`-store` 参数指定了使用哪种类型的存储，在这个场景下是 `file`. 另一个参数`-dir`则是指定应该存储在哪个目录。

在服务初次启动时，他将会在这个目录创建两个文件，一个包含服务器相关的信息\(`server.dat`\)，另一个则是记录客户端的信息\(`clients.dat`\)。

当一个streaming客户端连接时，他会使用服务注册在这个文件的客户端标识。当客户端断开连接时，文件会将客户端标识清除。

当客户端发布或者订阅新subject\(或者说channel\),服务将用这个subject的名字创建一个目录\(这也是为什么subject的名字不能包含/的原因\)。比如说，客户端订阅了`foo`,并且你的服务启动时指定了`-dir datastore`，那么你会发现一个`datastore/foo`的目录。在这个目录你会发现几个文件: 一个记录里subject的信息 \(`subs.dat`\)，另外一些则是消息的日志（`msgs.1.dat`）等等。

子目录的数量和channel的数量是对应的，可以通过配置参数`-max_channels` 来限制。一旦达到了限制，新的channel的新增订阅和消息的发布都会报错。

在指定的channel中，订阅的数量也可以通过配置参数来`-max_subs`限制。当客户端订阅一个订阅数量达到限制的channel时会返回异常。

Finally, the number of stored messages for a given channel can also be limited with the parameter `-max_msgs` and/or `-max_bytes`. However, for messages, the client does not get an error when the limit is reached. The oldest messages are discarded to make room for the new messages.

最后，还可以通过 `-max_msgs`和 `-max_bytes`来限制指定的channel存储的消息的数量。但是，当达到消息数量限制的时，客户端发送消息并不会报错，旧的消息将被抛弃，

## 选项

As described in the [Configuring](../cfgfile.md#configuration-file) section, there are several options that you can use to configure a file store.

Regardless of channel limits, you can configure message logs to be split in individual files \(called file slices\). You can configure those slices by number of messages it can contain \(`--file_slice_max_msgs`\), the size of the file - including the corresponding index file \(`--file_slice_max_bytes`\), or the period of time that a file slice should cover - starting at the time the first message is stored in that slice \(`--file_slice_max_age`\). The default file store options are defined such that only the slice size is configured to 64MB.

> **Note**: If you don't configure any slice limit but you do configure channel limits, then the server will automatically set some limits for file slices.

When messages accumulate in a channel, and limits are reached, older messages are removed. When the first file slice becomes empty, the server removes this file slice \(and corresponding index file\).

However, if you specify a script \(`--file_slice_archive_script`\), then the server will rename the slice files \(data and index\) with a `.bak` extension and invoke the script with the channel name, data and index file names.  
The files are left in the channel's directory and therefore it is the script responsibility to delete those files when done. At any rate, those files will not be recovered on a server restart, but having lots of unused files in the directory may slow down the server restart.

For instance, suppose the server is about to delete file slice `datastore/foo/msgs.1.dat` \(and `datastore/foo/msgs.1.idx`\), and you have configured the script `/home/nats-streaming/archive_script.sh`. The server will invoke:

```bash
/home/nats-streaming/archive_script.sh foo datastore/foo/msgs.1.dat.bak datastore/foo/msgs.2.idx.bak
```

Notice how the files have been renamed with the `.bak` extension so that they are not going to be recovered if the script leave those files in place.

As previously described, each channel corresponds to a sub-directory that contains several files. It means that the need for file descriptors increase with the number of channels. In order to scale to ten or hundred thousands of channels, the option `fds_limit` \(or command line parameter `--file_fds_limit`\) may be considered to limit the total use of file descriptors.

Note that this is a soft limit. It is possible for the store to use more file descriptors than the given limit if the number of concurrent read/writes to different channels is more than the said limit. It is also understood that this may affect performance since files may need to be closed/re-opened as needed.

## 恢复异常

We have added the ability for the server to truncate any file that may otherwise report an `unexpected EOF` error during the recovery process.

Since dataloss is likely to occur, the default behavior for the server on startup is to report recovery error and stop. It will now print the content of the first corrupted record before exiting.

With the `-file_truncate_bad_eof` parameter, the server will still print those bad records but truncate each file at the position of the first corrupted record in order to successfully start.

To prevent the use of this parameter as the default value, this option is not available in the configuration file. Moreover, the server will fail to start if started more than once with that parameter.  
This flag may help recover from a store failure, but since data may be lost in that process, we think that the operator needs to be aware and make an informed decision.

Note that this flag will not help with file corruption due to bad CRC for instance. You have the option to disable CRC on recovery with the `-file_crc=false` option.

Let's review the impact and suggested steps for each of the server's corrupted files:

* `server.dat`: This file contains meta data and NATS subjects used to communicate with client applications. If a corruption is reported with this file, we would suggest that you stop all your clients, stop the server, remove this file, restart the server. This will create a new `server.dat` file, but will not attempt to recover the rest of the channels because the server assumes that there is no state. So you should stop and restart the server once more. Then, you can restart all your clients.
* `clients.dat`: This contains information about client connections. If the file is truncated to move past an `unexpected EOF` error, this can result in no issue at all, or in client connections not being recovered, which means that the server will not know about possible running clients, and therefore it will not try to deliver any message to those non recovered clients, or reject incoming published messages from those clients. It is also possible that the server recovers a client connection that was actually closed. In this case, the server may attempt to deliver or redeliver messages unnecessarily.
* `subs.dat`: This is a channel's subscriptions file \(under the channel's directory\). If this file is truncated and some records are lost, it may result in no issue at all, or in client applications not receiving their messages since the server will not know about them. It is also possible that acknowledged messages get redelivered \(since their ack may have been lost\).
* `msgs.<n>.dat`: This is a channel's message log \(several per channel\). If one of those files is truncated, then message loss occurs. With the `unexpected EOF` errors, it is likely that only the last "file slice" of a channel will be affected. Nevertheless, if a lower sequence file slice is truncated, then gaps in message sequence will occur. So it would be possible for a channel to have now messages 1..100, 110..300 for instance, with messages 101 to 109 missing. Again, this is unlikely since we expect the unexpected end-of-file errors to occur on the last slice.

For _Clustered_ mode, this flag would work only for the NATS Streaming specific store files. As you know, NATS Streaming uses RAFT for consensus, and RAFT uses its own logs. You could try the option if the server reports `unexpected EOF` errors for NATS Streaming file stores, however, you may want to simply delete all NATS Streaming and RAFT stores for the failed node and restart it. By design, the other nodes in the cluster have replicated the data, so this node will become a follower and catchup with the rest of the cluster, getting the data from the current leader and recreating its local stores.

