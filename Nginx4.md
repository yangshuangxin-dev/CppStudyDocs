# Nginx的handler开发

Nginx不仅仅可以方向代理进行请求的转发、负载均衡、过滤器，还可以直接处理浏览器的HTTP请求，直接返回数据给浏览器，其处理就是Nginx的handler模式。由于Nginx是主进程进行监听端口，具体的业务逻辑是在子进程worker中，如果有多个woker需要使用共享内存来保证数据的一致性。

> Nginx所有的listen端口都在Nginx的主进程中，业务处理都在worker进程中，为了避免"惊群"现象的发生，在listen的fd加入worker进程时会加锁，保证只有listen的fd只加入一个worker进程的epoll中，避免发生"惊群"现象，提高性能。

handler的开发大致的步骤有三步：

1. Nginx启动时申请需要的资源，进行初始化。
2. 读取解析conf文件，对自定义的命令字进行解析保存。
3. 当收到浏览器的请求后，进入handler处理流程，返回数据给浏览器。

以统计客户端请求的handler开发为例，介绍handler开发的流程。handler的开发第一步还是把模块加入Nginx的编译源码中，编写conf文件：

```shell
ngx_addon_name=ngx_http_pagecount_module
HTTP_MODULES="$HTTP_MODULES ngx_http_pagecount_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_pagecount_module.c"
```

在编译的时候增加模块编译的路径：

```shell
./configure --prefix=/usr/local/nginx --with-http_realip_module --with-http_addition_module --with-http_ssl_module --with-http_gzip_static_module --with-http_secure_link_module --with-http_stub_status_module --with-stream --with-pcre=/home/yangshuangxin/share/nginx/pcre-8.41 --with-zlib=/home/yangshuangxin/share/nginx/zlib-1.2.11 --with-openssl=/home/yangshuangxin/share/nginx/openssl-1.1.0g --add-module=/home/yangshuangxin/share/nginx/ngx_http_pagecount_module
make -j
make install
```

## 单进程局域网

做局域网中，可以只启动一个Nginx的工作进程进行数据的处理，不需要考虑多进程的数据共享，并且可以使用256的数组进行KV键值对的存储，逻辑简单清晰并且容易实现。

```c++
#include <arpa/inet.h>
#include <netinet/in.h>
#include <ngx_core.h>
#include <ngx_http.h>

// 256 个数组，以ip地址的最低位作为数组的下标
typedef struct {
  int count;
  struct in_addr addr; // ip地址
} ngx_pv_table_t;

ngx_pv_table_t pv_table[256];
// conf配置文件的处理函数
char *ngx_http_handler_count_set(ngx_conf_t *cf, ngx_command_t *cmd,
                                 void *conf);
// handler的处理，进行html的拼接
ngx_int_t ngx_http_handler_count_handler(ngx_http_request_t *r);

static ngx_command_t ngx_http_handler_count_cmds[] = {
    {ngx_string("yangshuangxin"), // 命令字
     NGX_HTTP_LOC_CONF |
         NGX_CONF_NOARGS,         // 命令字的位置，http下的loc位置，并且没有参数
     ngx_http_handler_count_set,  // 命令字的处理
     NGX_HTTP_LOC_CONF_OFFSET, 0, // 命令字没有参数，不需要保存
     NULL},
    ngx_null_command // 结束符
};

// 8 个配置文件conf的回调函数，不需要使用
static ngx_http_module_t ngx_http_handler_count_ctx = {
    NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL,
};

ngx_module_t ngx_http_handler_count_module = {
    NGX_MODULE_V1,
    &ngx_http_handler_count_ctx, // 设置回调函数
    ngx_http_handler_count_cmds, // 设置命令字
    NGX_HTTP_MODULE,             // 设置http模块
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING};

// nginx.conf --> yangshuangxin
char *ngx_http_handler_count_set(ngx_conf_t *cf, ngx_command_t *cmd,
                                 void *conf) {
  // init 获取模块
  ngx_http_core_loc_conf_t *corecf =
      ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
  // 设置handler处理函数
  corecf->handler = ngx_http_handler_count_handler;

  return NGX_CONF_OK;
}
// 组装html数据
static int ngx_http_encode_page(char *html) {

  sprintf(html, "<h1>yangshuangxin</h1>");
  strcat(html, "<h2>");

  int i = 0;
  for (i = 0; i < 256; i++) {
    if (pv_table[i].count != 0) {

      char str[INET_ADDRSTRLEN] = {0};
      char buffer[128] = {0};

      snprintf(buffer, 128, "req from : %s, count : %d <br/>",
               inet_ntop(AF_INET, &pv_table[i].addr, str, sizeof(str)),
               pv_table[i].count);

      strcat(html, buffer);
    }
  }

  strcat(html, "<h2/>");

  return 0;
}

ngx_int_t ngx_http_handler_count_handler(ngx_http_request_t *r) {
  u_char html[1024] = {0};
  int len = sizeof(html);
  struct sockaddr_in *client_addr =
      (struct sockaddr_in *)r->connection->sockaddr; // 取到ip地址

  int idx =
      client_addr->sin_addr.s_addr >> 24; // 取到IP地址的最后8位作为数组下标
  pv_table[idx].count++;
  memcpy(&pv_table[idx].addr, &client_addr->sin_addr,
         sizeof(client_addr->sin_addr));

  ngx_http_encode_page((char *)html); // 组装html

  // 设置http的header，200 OK，类型的html
  r->headers_out.status = 200;
  ngx_str_set(&r->headers_out.content_type, "text/html");
  ngx_http_send_header(r);

  // 设置http请求的body
  ngx_buf_t *b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));

  ngx_chain_t out;
  out.buf = b;
  out.next = NULL;

  b->pos = html;
  b->last = html + len;
  b->memory = 1;
  b->last_buf = 1;
  // 回复请求
  return ngx_http_output_filter(r, &out);
}
```

## 多进程共享内存红黑树

需要处理的IP有很多，需要使用多个worker子进程进行数据的处理，并且256个数量太少不满足需求，需要使用共享内存的红黑树，增加了可记录的数量并且保证在多进程之间数据的一致性。

```c++
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>
// 命令字的处理函数，申请共享内存
static char *ngx_http_pagecount_set(ngx_conf_t *cf, ngx_command_t *cmd,
                                    void *conf);
// handler处理函数
static ngx_int_t ngx_http_pagecount_handler(ngx_http_request_t *r);
// 进入conf中http的配置文件的回调函数，申请共享保存内存的结构体
static void *ngx_http_pagecount_create_location_conf(ngx_conf_t *cf);
// 共享内存初始化，对共享内存进行分块（slab），切分为一个个红黑树的节点
static ngx_int_t ngx_http_pagecount_shm_init(ngx_shm_zone_t *zone, void *data);
// 红黑树的节点比较函数
static void ngx_http_pagecount_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                                   ngx_rbtree_node_t *node,
                                                   ngx_rbtree_node_t *sentinel);

static ngx_command_t count_commands[] = {
    {ngx_string("yangshuangxin"),         // 命令字
     NGX_HTTP_LOC_CONF | NGX_CONF_NOARGS, // 命令字的位置
     ngx_http_pagecount_set,              // 命令字的处理函数
     NGX_HTTP_LOC_CONF_OFFSET, 0,         // 命令字无参数，不需要申请内存
     NULL},
    ngx_null_command // 结束符，空命令字
};

static ngx_http_module_t count_ctx = {
    // conf全局的回调函数
    NULL,
    NULL,
    // 协议（HTTP）的回调函数
    NULL,
    NULL,
    // server的回调函数
    NULL,
    NULL,
    // location的回调函数
    ngx_http_pagecount_create_location_conf,
    NULL,
};

// ngx_http_count_module
ngx_module_t ngx_http_pagecount_module = {NGX_MODULE_V1,   // 模块固定头
                                          &count_ctx,      // 设置回调函数
                                          count_commands,  // 设置命令字
                                          NGX_HTTP_MODULE, // 设置位置，http模块
                                          NULL,
                                          NULL,
                                          NULL,
                                          NULL,
                                          NULL,
                                          NULL,
                                          NULL,
                                          NGX_MODULE_V1_PADDING};

// 红黑树
typedef struct {
  ngx_rbtree_t rbtree;        // 红黑树的根节点
  ngx_rbtree_node_t sentinel; // 红黑树的哨兵节点
} ngx_http_pagecount_shm_t;

// 共享内存结构
typedef struct {
  ssize_t shmsize;
  ngx_slab_pool_t *shpool;
  ngx_http_pagecount_shm_t *sh;
} ngx_http_pagecount_conf_t;

// 红黑树比较和插入函数
static void
ngx_http_pagecount_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                       ngx_rbtree_node_t *node,
                                       ngx_rbtree_node_t *sentinel) {
  ngx_rbtree_node_t **p;
  for (;;) {
    if (node->key < temp->key) {
      p = &temp->left;
    } else if (node->key > temp->key) {
      p = &temp->right;
    } else {
      return;
    }

    if (*p == sentinel) {
      break;
    }

    temp = *p;
  }

  *p = node;

  node->parent = temp;
  node->left = sentinel;
  node->right = sentinel;
  ngx_rbt_red(node);
}

// 初始化共享内存，把共享内存切分为红黑树节点的块slab
ngx_int_t ngx_http_pagecount_shm_init(ngx_shm_zone_t *zone, void *data) {
  ngx_http_pagecount_conf_t *conf;
  ngx_http_pagecount_conf_t *oconf = data;
  conf = (ngx_http_pagecount_conf_t *)zone->data;

  if (oconf) {
    conf->sh = oconf->sh;
    conf->shpool = oconf->shpool;
    return NGX_OK;
  }

  // 申请划分共享内存的红黑树节点
  conf->shpool = (ngx_slab_pool_t *)zone->shm.addr;
  conf->sh = ngx_slab_alloc(conf->shpool, sizeof(ngx_http_pagecount_shm_t));
  if (conf->sh == NULL) {
    return NGX_ERROR;
  }

  conf->shpool->data = conf->sh;

  // 红黑树初始化，传入红黑树的根节点、哨兵节点、红黑树比较
  ngx_rbtree_init(&conf->sh->rbtree, &conf->sh->sentinel,
                  ngx_http_pagecount_rbtree_insert_value);

  return NGX_OK;
}

// 进入location的回调函数，申请共享内存结构体的内存
void *ngx_http_pagecount_create_location_conf(ngx_conf_t *cf) {
  ngx_http_pagecount_conf_t *conf;
  conf = ngx_palloc(cf->pool, sizeof(ngx_http_pagecount_conf_t));
  if (NULL == conf) {
    return NULL;
  }
  conf->shmsize = 0;
  return conf;
}

// 命令字的处理函数，对共享内存进行初始化和分块slab
static char *ngx_http_pagecount_set(ngx_conf_t *cf, ngx_command_t *cmd,
                                    void *conf) {
  ngx_shm_zone_t *shm_zone;
  ngx_str_t name = ngx_string("pagecount_slab_shm"); // 共享内存的名字

  ngx_http_pagecount_conf_t *mconf = (ngx_http_pagecount_conf_t *)
      conf; // 进入location的回调函数时申请的共享内存结构体
  ngx_http_core_loc_conf_t *corecf;

  mconf->shmsize = 1024 * 1024;
  // 共享内存挂在模块上
  shm_zone = ngx_shared_memory_add(cf, &name, mconf->shmsize,
                                   &ngx_http_pagecount_module);
  if (NULL == shm_zone) {
    return NGX_CONF_ERROR;
  }
  // 设置共享内存分块初始化分块的函数
  shm_zone->init = ngx_http_pagecount_shm_init;
  shm_zone->data = mconf;
  // 设置handler处理函数
  corecf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
  corecf->handler = ngx_http_pagecount_handler;

  return NGX_CONF_OK;
}
// 红黑树查找，增加点击次数
static ngx_int_t ngx_http_pagecount_lookup(ngx_http_request_t *r,
                                           ngx_http_pagecount_conf_t *conf,
                                           ngx_uint_t key) {

  ngx_rbtree_node_t *node, *sentinel;

  node = conf->sh->rbtree.root;
  sentinel = conf->sh->rbtree.sentinel;

  ngx_log_error(NGX_LOG_EMERG, r->connection->log, ngx_errno,
                " ngx_http_pagecount_lookup 111 --> %x\n", key);

  while (node != sentinel) {
    if (key < node->key) {
      node = node->left;
      continue;
    } else if (key > node->key) {
      node = node->right;
      continue;
    } else { // key == node
      node->data++;
      return NGX_OK;
    }
  }

  ngx_log_error(NGX_LOG_EMERG, r->connection->log, ngx_errno,
                " ngx_http_pagecount_lookup 222 --> %x\n", key);

  // 没有找到，分配slab内存，插入红黑树
  node = ngx_slab_alloc_locked(conf->shpool, sizeof(ngx_rbtree_node_t));
  if (NULL == node) {
    return NGX_ERROR;
  }

  node->key = key;
  node->data = 1;
  ngx_rbtree_insert(&conf->sh->rbtree, node);
  ngx_log_error(NGX_LOG_EMERG, r->connection->log, ngx_errno,
                " insert success\n");

  return NGX_OK;
}
// 遍历红黑树，组装html数据
static int ngx_encode_http_page_rb(ngx_http_pagecount_conf_t *conf,
                                   char *html) {

  sprintf(html, "<h1>yangshuangxin</h1>");

  strcat(html, "<h2>");
  ngx_rbtree_node_t *node =
      ngx_rbtree_min(conf->sh->rbtree.root, conf->sh->rbtree.sentinel);

  do {
    char str[INET_ADDRSTRLEN] = {0};
    char buffer[128] = {0};
    sprintf(buffer, "req from : %s, count: %d <br/>",
            inet_ntop(AF_INET, &node->key, str, sizeof(str)), node->data);

    strcat(html, buffer);
    node = ngx_rbtree_next(&conf->sh->rbtree, node);
  } while (node);
  strcat(html, "</h2>");

  return NGX_OK;
}
// handler处理函数
static ngx_int_t ngx_http_pagecount_handler(ngx_http_request_t *r) {
  // html页面数据
  u_char html[1024] = {0};
  int len = sizeof(html);

  ngx_rbtree_key_t key = 0;
  // 获取到客户端的ip地址
  struct sockaddr_in *client_addr =
      (struct sockaddr_in *)r->connection->sockaddr;
  // 取到共享内存的结构体
  ngx_http_pagecount_conf_t *conf =
      ngx_http_get_module_loc_conf(r, ngx_http_pagecount_module);
  // ip地址作为红黑树的KEY
  key = (ngx_rbtree_key_t)client_addr->sin_addr.s_addr;

  ngx_log_error(NGX_LOG_EMERG, r->connection->log, ngx_errno,
                " ngx_http_pagecount_handler --> %x\n", key);
  // 操作共享内存，增加访问次数
  ngx_shmtx_lock(&conf->shpool->mutex);
  ngx_http_pagecount_lookup(r, conf, key);
  ngx_shmtx_unlock(&conf->shpool->mutex);

  ngx_encode_http_page_rb(conf, (char *)html);

  // 组装http的header
  r->headers_out.status = 200;
  ngx_str_set(&r->headers_out.content_type, "text/html");
  ngx_http_send_header(r);

  // 组装http的body
  ngx_buf_t *b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));

  ngx_chain_t out;
  out.buf = b;
  out.next = NULL;

  b->pos = html;
  b->last = html + len;
  b->memory = 1;
  b->last_buf = 1;
  // 数据返回客户端
  return ngx_http_output_filter(r, &out);
}

```

