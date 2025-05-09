# Nginx过滤器的开发

Nginx会对浏览器的的HTTP请求进行接收，接收后Nginx可以进行转发给其他server进行处理，当其他server处理后的数据再通过Nginx返回给浏览器。在Nginx返回数据给浏览器的时候，Nginx可以对其中的数据进行过滤和修改，这样就可以不用修改后端的业务逻辑，对所有返回的数据增加自定义的内容，例如增加广告。

Nginx的过滤器开发需要增加Nginx的http模块，当Nginx编译时增加到Nginx编译的源码中，和Nginx一起进行编译。增加Nginx的编译模块需要增加config文件，把需要编译的源码增加到Nginx源码编译中。

```shell
ngx_addon_name=ngx_http_prefix_filter_module
HTTP_FILTER_MODULES="$HTTP_FILTER_MODULES ngx_http_prefix_filter_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_prefix_filter_module.c"
```

当进行编译时，指定过滤器模块的路径：

```shell
./configure --prefix=/usr/local/nginx --with-http_realip_module --with-http_addition_module --with-http_ssl_module --with-http_gzip_static_module --with-http_secure_link_module --with-http_stub_status_module --with-stream --with-pcre=/home/yangshuangxin/share/nginx/pcre-8.41 --with-zlib=/home/yangshuangxin/share/nginx/zlib-1.2.11 --with-openssl=/home/yangshuangxin/share/nginx/openssl-1.1.0g --add-module=/home/yangshuangxin/share/nginx/ngx_http_prefix_filter_module
make -j 
make install
```

创建ngx_http_prefix_filter_module.c过滤器源代码，和config放到同一个路径下。

```c++
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>
// 从Nginx配置文件中读取到过滤器是否启动的开关
typedef struct {
  ngx_flag_t enable;
} ngx_http_prefix_filter_conf_t;
// http模块初始化时的回调函数，需要把过滤器模块插入到http模块的处理链表中
static ngx_int_t ngx_http_prefix_filter_init(ngx_conf_t *cf);
// 对http协议头增加过滤器的修改的长度值
static ngx_int_t ngx_http_prefix_filter_header_filter(ngx_http_request_t *r);
// 对http协议的内容增加自定义修改的内容
static ngx_int_t ngx_http_prefix_filter_body_filter(ngx_http_request_t *r,
                                                    ngx_chain_t *in);
// 过滤器自定义的内存，所有的http返回值都会增加该值
static ngx_str_t filter_prefix = ngx_string(
    "<h2>Author : yangshuangxin</h2><p><a "
    "href=\"https://www.zhihu.com/people/yangshuangxin\">yangshuangxin</a></p>");

// 进入配置Nginx的配置文件时调用，申请配置的内存
static void *ngx_http_prefix_filter_create_conf(ngx_conf_t *cf) {
  ngx_http_prefix_filter_conf_t *conf =
      ngx_pcalloc(cf->pool, sizeof(ngx_http_prefix_filter_conf_t));
  if (conf == NULL) {
    return NULL;
  }

  conf->enable = NGX_CONF_UNSET;

  return conf;
}
// 当读取到Nginx的配置文件的add_prefix字段，对ngx_http_prefix_filter_conf_t的
// enable 进行赋值
static ngx_command_t ngx_http_prefix_filter_commands[] = {
    {ngx_string("add_prefix"), // 配置文件的配置名称
     NGX_HTTP_MAIN_CONF | NGX_HTTP_SRV_CONF | NGX_HTTP_LOC_CONF |
         NGX_HTTP_LMT_CONF | NGX_CONF_FLAG, // add_prefix 可以存在的位置
     ngx_conf_set_flag_slot,                // 处理函数
     NGX_HTTP_LOC_CONF_OFFSET,              // loc的偏移值
     offsetof(ngx_http_prefix_filter_conf_t,
              enable), // enable相对ngx_http_prefix_filter_conf_t的偏移量
     NULL},
    ngx_null_command // 命令字结束标志，必须增加
};
// 处理完Nginx所有的配置文件中的配置项，需要把过滤器的配置合入到所有全局的配置项中
static char *ngx_http_prefix_filter_merge_conf(ngx_conf_t *cf, void *parent,
                                               void *child) {
  ngx_http_prefix_filter_conf_t *prev = (ngx_http_prefix_filter_conf_t *)parent;
  ngx_http_prefix_filter_conf_t *conf = (ngx_http_prefix_filter_conf_t *)child;

  ngx_conf_merge_value(conf->enable, prev->enable, 0);

  return NGX_CONF_OK;
}

// 过滤器模块自定义上下文，用来控制http响应增加自定义内容
typedef struct {
  ngx_int_t add_prefix;
} ngx_http_prefix_filter_ctx_t;

// http过滤器模块的上下文，设置8个回调函数
static ngx_http_module_t ngx_http_prefix_filter_module_ctx = {
	//conf
    NULL,
    ngx_http_prefix_filter_init,
	// main
    NULL,
    NULL,
	 //server
    NULL,
    NULL,
	 // loc
    ngx_http_prefix_filter_create_conf,
    ngx_http_prefix_filter_merge_conf};

// 定义过滤器的模块，设置回调函数、配置文件的命令字、模块的位置
ngx_module_t ngx_http_prefix_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_prefix_filter_module_ctx,
    ngx_http_prefix_filter_commands,
    NGX_HTTP_MODULE,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING};

// 使用头插法，把过滤器模块插入HTTP处理的模块中
static ngx_http_output_header_filter_pt ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt ngx_http_next_body_filter;

static ngx_int_t ngx_http_prefix_filter_init(ngx_conf_t *cf) {

  ngx_http_next_header_filter = ngx_http_top_header_filter;
  ngx_http_top_header_filter = ngx_http_prefix_filter_header_filter;

  ngx_http_next_body_filter = ngx_http_top_body_filter;
  ngx_http_top_body_filter = ngx_http_prefix_filter_body_filter;

  return NGX_OK;
}

// 过滤器对http请求头增加长度
static ngx_int_t ngx_http_prefix_filter_header_filter(ngx_http_request_t *r) {

  ngx_http_prefix_filter_ctx_t *ctx;
  ngx_http_prefix_filter_conf_t *conf;

  if (r->headers_out.status != NGX_HTTP_OK) {
    return ngx_http_next_header_filter(r);
  }
// 查找是否存在过滤器模块，过滤器是否开启
  ctx = ngx_http_get_module_ctx(r, ngx_http_prefix_filter_module);
  if (ctx) {
    return ngx_http_next_header_filter(r);
  }

  conf = ngx_http_get_module_loc_conf(r, ngx_http_prefix_filter_module);
  if (conf == NULL) {
    return ngx_http_next_header_filter(r);
  }
  if (conf->enable == 0) {
    return ngx_http_next_header_filter(r);
  }
// 分配自定义上下文，控制增加http请求的head和body
  ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_prefix_filter_ctx_t));
  if (ctx == NULL) {
    return NGX_ERROR;
  }
  ctx->add_prefix = 0;

  ngx_http_set_ctx(r, ctx, ngx_http_prefix_filter_module);
// 增加http请求的头长度
  if (r->headers_out.content_type.len >= sizeof("text/html") - 1 &&
      ngx_strncasecmp(r->headers_out.content_type.data, (u_char *)"text/html",
                      sizeof("text/html") - 1) == 0) {

    ctx->add_prefix = 1;
    if (r->headers_out.content_length_n > 0) {
      r->headers_out.content_length_n += filter_prefix.len;
    }
  }

  return ngx_http_prefix_filter_header_filter(r);
}
// 增加http请求体内容
static ngx_int_t ngx_http_prefix_filter_body_filter(ngx_http_request_t *r,
                                                    ngx_chain_t *in) {
// 查找是否存在过滤器，并且已经增加了head的长度
  ngx_http_prefix_filter_ctx_t *ctx =
      ngx_http_get_module_ctx(r, ngx_http_prefix_filter_module);
  if (ctx == NULL || ctx->add_prefix != 1) {
    return ngx_http_next_body_filter(r, in);
  }

  ctx->add_prefix = 2;
// 对body新增内容
  ngx_buf_t *b = ngx_create_temp_buf(r->pool, filter_prefix.len);
  b->start = b->pos = filter_prefix.data;
  b->last = b->pos + filter_prefix.len;

  ngx_chain_t *cl = ngx_alloc_chain_link(r->pool);
  cl->buf = b;
  cl->next = in;

  return ngx_http_next_body_filter(r, cl);
}

```

