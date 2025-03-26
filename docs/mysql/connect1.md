# 当客户端输入 mysql -h x.x.x.x -u root -p 时，服务端在做什么（上）

!!! note "Conventions"

    - [x] 每一个代码块的顶部都有它所属的文件（相对）路径，如果代码块属于某个函数（方法），那么顶部会有函数（方法）的声明。
    - [x] 代码块中的对象，如果很重要，会在行尾补充该对象的声明。
    - [x] 我使用的MySQL 版本是 8.0.41。你看到这篇时，可能有了更新的版本，比如 8.0.42，区别不大的。

---

## 等待客户端连接

服务端在启动后监听 3306 端口，并且一直等待客户端连接。对应到源代码是什么样子的？

``` cpp title="sql/main.cc" linenums="24"
extern int mysqld_main(int argc, char **argv);

int main(int argc, char **argv) { return mysqld_main(argc, argv); }
```

首先，main 函数直接调用 mysqld_main 函数，够简单吧，那就去看看 mysqld_main 函数。

``` cpp title="sql/mysqld.cc int mysqld_main(int argc, char **argv)" linenums="8286"
  mysqld_socket_acceptor->connection_event_loop(); // (1)!
```
mysqld_main 函数太复杂了，别怕，大部分我们都不关心，我们只抓主要矛盾。直接看 8286 行，mysqld_socket_acceptor 调用 connection_event_loop 函数负责等待客户端的连接事件。

1.  static Connection_acceptor<Mysqld_socket_listener> *mysqld_socket_acceptor

``` cpp title="sql/conn_handler/connection_acceptor.h" linenums="58" hl_lines="8 9"
  /**
    Connection acceptor loop to accept connections from clients.
  */
  void connection_event_loop() {
    Connection_handler_manager *mgr =
        Connection_handler_manager::get_instance();
    while (!connection_events_loop_aborted()) {
      Channel_info *channel_info = m_listener->listen_for_connection_event(); // (1)!
      if (channel_info != nullptr) mgr->process_new_connection(channel_info);
    }
  }
```

1.  Mysqld_socket_listener *m_listener

connection_event_loop() 函数没几行，从第 65 行看出，mysqld_socket_acceptor 的成员 m_listener 才是真正干活的———等待连接事件。第 66 行将（封装好的）连接对象 channel_info 交给 mgr 对象，mgr 对象调用 process_new_connection() 函数处理连接，就像它的名字 Connection_handler_manager 所说的，它是连接句柄的管理者。

``` cpp title="sql/conn_handler/socket_connection.cc" linenums="1348" hl_lines="3"
Channel_info *Mysqld_socket_listener::listen_for_connection_event() {
#ifdef HAVE_POLL
  int retval = poll(&m_poll_info.m_fds[0], m_socket_vector.size(), -1);
#else
  m_select_info.m_read_fds = m_select_info.m_client_fds;
  int retval = select((int)m_select_info.m_max_used_connection,
                      &m_select_info.m_read_fds, 0, 0, 0);
#endif
```

苦力 m_listener 使用 POLL 系统调用，监听 POLLIN 事件。

``` cpp title="sql/conn_handler/socket_connection.cc Channel_info *listen_for_connection_event()" linenums="1419" hl_lines="5-7"
  Channel_info *channel_info = nullptr;
  if (listen_socket->m_socket_type == Socket_type::UNIX_SOCKET)
    channel_info = new (std::nothrow) Channel_info_local_socket(connect_sock);
  else
    channel_info = new (std::nothrow) Channel_info_tcpip_socket(
        connect_sock, (listen_socket->m_socket_interface ==
                       Socket_interface_type::ADMIN_INTERFACE));
#endif
```

接着，POLLIN 事件发生后，new 一个 channel_info 对象，类型 Channel_info_tcpip_socket，listen_for_connection_event 函数返回的就是这个对象。

``` cpp title="sql/conn_handler/connection_handler_manager.cc" linenums="254" hl_lines="10"
void Connection_handler_manager::process_new_connection(
    Channel_info *channel_info) {
  if (connection_events_loop_aborted() ||
      !check_and_incr_conn_count(channel_info->is_admin_connection())) {
    channel_info->send_error_and_close_channel(ER_CON_COUNT_ERROR, 0, true);
    delete channel_info;
    return;
  }

  if (m_connection_handler->add_connection(channel_info)) { // (1)!
    inc_aborted_connects();
    delete channel_info;
  }
}
```

1.  Connection_handler *m_connection_handler

## 处理连接

前面说到 mgr 调用 process_new_connection() 函数处理具体的连接对象 channel_info，上面就是该函数的全部代码，直接看第 263 行，它把连接对象 channel_info 交给了 m_connection_handler，m_connection_handler 是 mgr 的成员，对应类型 Connection_handler。

``` cpp title="sql/conn_handler/connection_handler_per_thread.cc bool Per_thread_connection_handler::add_connection(Channel_info *channel_info)" linenums="407" hl_lines="1 9 10"
  if (!check_idle_thread_and_enqueue_connection(channel_info)) return false;

  /*
    There are no idle threads available to take up the new
    connection. Create a new thread to handle the connection
  */
  channel_info->set_prior_thr_create_utime();
  error =
      mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                          handle_connection, (void *)channel_info);
```

add_connection 函数只需要关注上面几行代码。

Connection_handler 有两个子类，一是 Per_thread_connection_handler，另一个是 One_thread_connection_handler。顾名思义，Per_thread_connection_handler 是一个连接一个线程，One_thread_connection_handler 是所有连接一个线程。我们看 Per_thread_connection_handler 就可以了，因为这是默认也是正常情况下用的，如果你一定要用 One_thread_connection_handler，那么可以修改 MySQL 配置 thread_handling=no-threads 即可。

从这里我们知道了，MySQL 服务端对于每一个客户端连接，都会创建一个线程来处理。按照套路，肯定得有线程池，第 407 行的函数名 check_idle_thread_and_enqueue_connection 就在提示我们，检查有没有空闲的线程，如果有那么就把连接对象 channel_info 扔进去。

我很自然地想到了线程池最大线程数的配置，于是查找[官方文档](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_cache_size)，找到了 thread_cache_size。

扯远了，我们接着看上面这个函数，由于我们是服务端启动后第一次连接，所以 check_idle_thread_and_enqueue_connection 会返回 true 表示没有空闲线程，程序会走到第 415、416 行，mysql_thread_create 函数创建线程（内部使用 pthread API），并且将 handle_connection 函数指示符作为参数，这样线程就会执行 handle_connection 函数，而 (void *)channel_info 最终会作为 handle_connection 函数的入参。

``` cpp title="sql/conn_handler/connection_handler_per_thread.cc static void *handle_connection(void *arg)" linenums="299" hl_lines="1"
    if (thd_prepare_connection(thd))
      handler_manager->inc_aborted_connects();
    else {
      while (thd_connection_alive(thd)) {
        if (do_command(thd)) break;
      }
      end_connection(thd);
    }
```

线程执行的 handle_connection 函数如上所示，我们只需要关注这么一小块就行了。第 299 行 thd_prepare_connection 会验证用户名和密码，如果正确，那么返回 false，否则返回 true。

假设用户名密码正确，即 thd_prepare_connection 返回 false，程序会进入 else 分支。else 分支中是一个 while 循环，while 判断当连接存活时，调用 do_command 函数；连接不存活时，跳出循环，调用 end_connection 函数。

do_command 函数会阻塞等待客户端的下一条命令。

!!! tip

    MySQL 源代码中很多函数成功时返回 false，失败时返回 true，要看注释。 

---

## 小结

- 类 `Connection_acceptor<Mysqld_socket_listener>` 负责监听 3306 端口，与客户端建立连接。
- 类 `Mysqld_socket_listener` 是实际与客户端建立网络（TCP/IP）连接的牛马。
- 类 `Channel_info` 封装连接信息，子类 `Channel_info_tcpip_socket` 表示 TCP/IP 连接。
- 类 `Connection_handler` 负责处理连接，子类 `Per_thread_connection_handler` 对于每一个客户端连接，都会新建一个线程，处理逻辑在 `handle_connection` 方法中。