# Spdk的文件系统开发

在Posix中，文件的读写操作都是同步的，而Spdk中几乎所有的接口都是异步处理，都有一个完成某一个操作后的回调函数。想要兼容替换Posix的文件操作，首先就需要把Spdk的异步操作改成同步操作。

Posix的基本读写操作如下：

```c++
int main() {
    // 写
	int fd = open("ysx.txt", O_RDWR | O_CREAT);
	char *wbuffer = "yangshuangxin";
	write(fd, wbuffer, strlen(wbuffer));
	close(fd);
	// 读
	int fd = open("ysx.txt", O_RDWR | O_CREAT);
	char rbuffer[1024] = {0};
	read(fd, rbuffer, 1024);
	close(fd);
}

```

Spdk改造成同步操作，需要使用一个消息线程进行等待直到在回调函数中完成：

```c++
#include <spdk/event.h>
#include <stdio.h>
#include <spdk/bdev.h>
#include <spdk/blob.h>
#include <spdk/env.h>
#include <spdk/blob_bdev.h>

#define FILENAME_LENGTH 128
// 各个回调函数传递的上下文
typedef struct ysxfs_context_s {

  struct spdk_bs_dev *bsdev;
  struct spdk_blob_store *blobstore;
  spdk_blob_id blobid;
  struct spdk_blob *blob;
  struct spdk_io_channel *channel;
  uint8_t *write_buffer;
  uint8_t *read_buffer;
  uint64_t io_unit_size;
  bool finished;
} ysxfs_context_t;

// 创建的同步等待线程
struct spdk_thread *global_thread = NULL;

static void ysxfs_bdev_event_call(enum spdk_bdev_event_type type,
                                  struct spdk_bdev *bdev, void *event_ctx) {
  SPDK_NOTICELOG("%s --> enter\n", __func__);
}

static const int POLLER_MAX_TIME = 100000;
// 等待异步完成
static bool poller(struct spdk_thread *thread, spdk_msg_fn start_fn, void *ctx,
                   bool *finished) {
  spdk_thread_send_msg(thread, start_fn, ctx);
  int poller_count = 0;
  do {
    spdk_thread_poll(thread, 0, 0);
    poller_count++;
  } while (!(*finished) && poller_count < POLLER_MAX_TIME);

  if (!(*finished) && poller_count >= POLLER_MAX_TIME) {
    return false;
  }
  return true;
}

static void ysxfs_bs_unload_complete(void *arg, int bserrno) {
  spdk_app_stop(1);
}
// 去初始化
static void ysxfs_bs_unload(ysxfs_context_t *ctx) {
  if (ctx->blobstore) {
    if (ctx->channel) {
      spdk_bs_free_io_channel(ctx->channel);
    }
    if (ctx->read_buffer) {
      spdk_free(ctx->read_buffer);
      ctx->read_buffer = NULL;
    }
    if (ctx->write_buffer) {
      spdk_free(ctx->write_buffer);
      ctx->write_buffer = NULL;
    }

    spdk_bs_unload(ctx->blobstore, ysxfs_bs_unload_complete, ctx);
  }
}

// 异步读使用poll改成同步读
static void ysxfs_blob_read_complete(void *arg, int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  SPDK_NOTICELOG("size: %ld, buffer: %s\n", ctx->io_unit_size,
                 ctx->read_buffer);
  ctx->finished = true;
}

static void ysxfs_do_read(void *arg) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  ctx->read_buffer = spdk_malloc(ctx->io_unit_size, 0x1000, NULL,
                                 SPDK_ENV_LCORE_ID_ANY, SPDK_MALLOC_DMA);
  if (ctx->read_buffer == NULL) {
    ysxfs_bs_unload(ctx);
    return;
  }
  memset(ctx->read_buffer, '\0', ctx->io_unit_size);

  spdk_blob_io_read(ctx->blob, ctx->channel, ctx->read_buffer, 0, 1,
                    ysxfs_blob_read_complete, ctx);
}

static void ysxfs_file_read(ysxfs_context_t *ctx) {
  ctx->finished = false;
  poller(global_thread, ysxfs_do_read, ctx, &ctx->finished);
}

// 异步写使用poll改成同步写 
static void ysxfs_blob_write_complete(void *arg, int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->finished = true;
}

static void ysxfs_do_write(void *arg) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->write_buffer = spdk_malloc(ctx->io_unit_size, 0x1000, NULL,
                                  SPDK_ENV_LCORE_ID_ANY, SPDK_MALLOC_DMA);
  if (ctx->write_buffer == NULL) {
    ysxfs_bs_unload(ctx);
    return;
  }

  memset(ctx->write_buffer, '\0', ctx->io_unit_size);
  memset(ctx->write_buffer, 'A', ctx->io_unit_size - 1);

  SPDK_NOTICELOG("%s --> enter \n", __func__);

  struct spdk_io_channel *channel = spdk_bs_alloc_io_channel(ctx->blobstore);
  if (channel == NULL) {
    ysxfs_bs_unload(ctx);
    return;
  }
  ctx->channel = channel;

  spdk_blob_io_write(ctx->blob, ctx->channel, ctx->write_buffer, 0, 1,
                     ysxfs_blob_write_complete, ctx);
}

static void ysxfs_file_write(ysxfs_context_t *ctx) {
  ctx->finished = false;
  poller(global_thread, ysxfs_do_write, ctx, &ctx->finished);
}

// blob resize 后，设置完成标志
static void ysxfs_blob_sync_complete(void *arg, int bserrno) {

  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->finished = true;
  SPDK_NOTICELOG("%s --> %lu enter\n", __func__, ctx->io_unit_size);
}

static void ysxfs_blob_resize_complete(void *arg, int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  spdk_blob_sync_md(ctx->blob, ysxfs_blob_sync_complete, ctx);
}

static void ysxfs_blob_open_complete(void *arg, struct spdk_blob *blob,
                                     int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->blob = blob;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  uint64_t freed = spdk_bs_free_cluster_count(ctx->blobstore);
  spdk_blob_resize(blob, freed, ysxfs_blob_resize_complete, ctx);
}

static void ysxfs_bs_create_complete(void *arg, spdk_blob_id blobid,
                                     int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->blobid = blobid;
  spdk_bs_open_blob(ctx->blobstore, blobid, ysxfs_blob_open_complete, ctx);
}

static void ysxfs_bs_init_complete(void *arg, struct spdk_blob_store *bs,
                                   int bserrno) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  ctx->blobstore = bs;
  ctx->io_unit_size = spdk_bs_get_io_unit_size(bs);
  SPDK_NOTICELOG("%s --> enter: %lu\n", __func__, ctx->io_unit_size);
  spdk_bs_create_blob(bs, ysxfs_bs_create_complete, ctx);
}

// 初始化，bdev->blobstore->blob->blob resize 后
static void ysxfs_entry(void *arg) {
  ysxfs_context_t *ctx = (ysxfs_context_t *)arg;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  const char *bdev_name = "Malloc0";
  int rc = spdk_bdev_create_bs_dev_ext(bdev_name, ysxfs_bdev_event_call, NULL,
                                       &ctx->bsdev);
  if (rc != 0) {
    spdk_app_stop(-1);
    return;
  }
  spdk_bs_init(ctx->bsdev, NULL, ysxfs_bs_init_complete, ctx);
}

static const char *json_file =
    "/home/spdk/examples/ysx/hello_blob.json";

static void json_app_load_done(int rc, void *ctx) {
  bool *done = ctx;
  *done = true;
}

static void ysxfs_json_load_fn(void *arg) {
  spdk_subsystem_init_from_json_config(json_file, SPDK_DEFAULT_RPC_ADDR,
                                       json_app_load_done, arg, true);
}

int main(int argc, char *argv[]) {

  printf("hello spdk\n");

  // 不能直接使用 spdk_app_start，因为需要手动创建spdk线程
  struct spdk_env_opts opts;
  spdk_env_opts_init(&opts);

  if (spdk_env_init(&opts) != 0) {
    return -1;
  }

  spdk_log_set_print_level(SPDK_LOG_NOTICE);
  spdk_log_set_level(SPDK_LOG_NOTICE);
  spdk_log_open(NULL);

  spdk_thread_lib_init(NULL, 0);
  global_thread = spdk_thread_create("global", NULL);
  spdk_set_thread(global_thread);

  bool done = false;
  poller(global_thread, ysxfs_json_load_fn, &done, &done);

  ysxfs_context_t *ctx = calloc(1, sizeof(ysxfs_context_t));
  if (ctx == NULL)
    return -1;
  memset(ctx, 0, sizeof(ysxfs_context_t));
  ctx->finished = false;
  // 初始化，直到blob resize回调函数完成
  poller(global_thread, ysxfs_entry, ctx, &ctx->finished);
  SPDK_NOTICELOG("--> ctx->io_unit_size: %ld\n", ctx->io_unit_size);
  // 同步写数据到blob
  ysxfs_file_write(ctx);
  // 同步读数据到buffer
  ysxfs_file_read(ctx);
  ysxfs_bs_unload(ctx);
  return 0;
}
```

把Spdk的封装成文件系统，首先需要定义Spdk的文件系统数据结构和文件的数据结构，文件系统是全局唯一的，可以作为单例或全局变量，只需要初始化一次。

```c++
#include <dlfcn.h>
#include <spdk/event.h>
#include <stdio.h>
#include <spdk/bdev.h>
#include <spdk/blob.h>
#include <spdk/env.h>
#include <spdk/blob_bdev.h>
#define FILENAME_LENGTH 128

typedef struct ysxfs_file_s {
  char filename[FILENAME_LENGTH];
  uint8_t *write_buffer;
  uint8_t *read_buffer;
  struct spdk_blob *blob;
  struct ysxfs_filesystem_s *fs;
} ysxfs_file_t;

typedef struct ysxfs_filesystem_s {
  struct spdk_bs_dev *bsdev;
  struct spdk_blob_store *blobstore;
  struct spdk_io_channel *channel;
  uint64_t io_unit_size;
  struct spdk_thread *thread;
  bool finished;

} ysxfs_filesystem_t;

ysxfs_filesystem_t *fs_instance = NULL;
```

文件系统的初始化和去初始化

```c++
// 初始化文件系统
static int ysxfs_filesystem_setup(void) {
  struct spdk_env_opts opts;
  spdk_env_opts_init(&opts);
  if (spdk_env_init(&opts) != 0) {
    return -1;
  }

  spdk_log_set_print_level(SPDK_LOG_NOTICE);
  spdk_log_set_level(SPDK_LOG_NOTICE);
  spdk_log_open(NULL);

  ysxfs_filesystem_t *fs = calloc(1, sizeof(ysxfs_filesystem_t));
  if (!fs) {
    return 0;
  }
  fs_instance = fs;

  spdk_thread_lib_init(NULL, 0);
  fs->thread = spdk_thread_create("global", NULL);
  spdk_set_thread(fs->thread);

  bool done = false; // load_config
  poller(fs->thread, ysxfs_json_load_fn, &done, &done);

  fs->finished = false; // filesystem_register;
  poller(fs->thread, ysxfs_entry, fs, &fs->finished);
  return 0;
}

static const char *json_file = "/home/spdk/examples/ysx/hello_blob.json";
static void json_app_load_done(int rc, void *ctx) {
  bool *done = ctx;
  *done = true;
}
static void ysxfs_json_load_fn(void *arg) {
  spdk_subsystem_init_from_json_config(json_file, SPDK_DEFAULT_RPC_ADDR,
                                       json_app_load_done, arg, true);
}

static void ysxfs_entry(void *arg) {
  ysxfs_filesystem_t *fs = (ysxfs_filesystem_t *)arg;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  const char *bdev_name = "Malloc0";
  int rc = spdk_bdev_create_bs_dev_ext(bdev_name, ysxfs_bdev_event_call, NULL,
                                       &fs->bsdev);
  if (rc != 0) {
    spdk_app_stop(-1);
    return;
  }

  spdk_bs_init(fs->bsdev, NULL, ysxfs_bs_init_complete, fs);
}
// blobstore、读写channel创建完成，就完成了文件系统初始化
static void ysxfs_bs_init_complete(void *arg, struct spdk_blob_store *bs,
                                   int bserrno) {
  ysxfs_filesystem_t *fs = (ysxfs_filesystem_t *)arg;
  fs->blobstore = bs;
  fs->io_unit_size = spdk_bs_get_io_unit_size(bs);
  struct spdk_io_channel *channel = spdk_bs_alloc_io_channel(fs->blobstore);
  if (channel == NULL) {
    ysxfs_bs_unload(fs);
    return;
  }
  fs->channel = channel;
  SPDK_NOTICELOG("%s --> enter: %lu\n", __func__, fs->io_unit_size);
  fs->finished = true;
}
// 去初始化文件系统
static void ysxfs_filesystem_unregister(ysxfs_filesystem_t *fs) {
  fs->finished = false;
  poller(fs->thread, ysxfs_bs_unload, fs, &fs->finished);
}

static void ysxfs_bs_unload(void *arg) {
  ysxfs_filesystem_t *fs = (ysxfs_filesystem_t *)arg;
  if (fs->blobstore) {
    if (fs->channel) {
      spdk_bs_free_io_channel(fs->channel);
    }
    spdk_bs_unload(fs->blobstore, ysxfs_bs_unload_complete, fs);
  }
}

static void ysxfs_bs_unload_complete(void *arg, int bserrno) {
  ysxfs_filesystem_t *fs = (ysxfs_filesystem_t *)arg;
  fs->finished = true;
  spdk_app_stop(1);
}
```

文件的创建操作，同步进行 创建blob、申请读写buffer的内存、blob进行大小的resize

```c++
static void ysxfs_file_create(ysxfs_file_t *file) {
  ysxfs_filesystem_t *fs = file->fs;
  fs->finished = false;
  poller(fs->thread, ysxfs_do_create, file, &fs->finished);
}

static void ysxfs_do_create(void *arg) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  spdk_bs_create_blob(fs->blobstore, ysxfs_bs_create_complete, file);
}

static void ysxfs_bs_create_complete(void *arg, spdk_blob_id blobid,
                                     int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  spdk_bs_open_blob(fs->blobstore, blobid, ysxfs_blob_open_complete, file);
}

static void ysxfs_blob_open_complete(void *arg, struct spdk_blob *blob,
                                     int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  file->blob = blob;

  SPDK_NOTICELOG("%s --> enter\n", __func__);
  uint64_t freed = spdk_bs_free_cluster_count(fs->blobstore);
  file->write_buffer = spdk_malloc(fs->io_unit_size, 0x1000, NULL,
                                   SPDK_ENV_LCORE_ID_ANY, SPDK_MALLOC_DMA);
  if (file->write_buffer == NULL) {
    return;
  }

  file->read_buffer = spdk_malloc(fs->io_unit_size, 0x1000, NULL,
                                  SPDK_ENV_LCORE_ID_ANY, SPDK_MALLOC_DMA);
  if (file->read_buffer == NULL) {
    spdk_free(file->write_buffer);
    return;
  }
  spdk_blob_resize(blob, freed, ysxfs_blob_resize_complete, file);
}

static void ysxfs_blob_resize_complete(void *arg, int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  spdk_blob_sync_md(file->blob, ysxfs_blob_sync_complete, file);
}

static void ysxfs_blob_sync_complete(void *arg, int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  fs->finished = true;
  SPDK_NOTICELOG("%s --> %lu enter\n", __func__, fs->io_unit_size);
}
```

文件创建成功后，可以进行写文件

```c++
static void ysxfs_file_write(ysxfs_file_t *file) {
  ysxfs_filesystem_t *fs = file->fs;
  fs->finished = false;
  poller(fs->thread, ysxfs_do_write, file, &fs->finished);
}

static void ysxfs_do_write(void *arg) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  spdk_blob_io_write(file->blob, fs->channel, file->write_buffer, 0, 1,
                     ysxfs_blob_write_complete, file);
}

static void ysxfs_blob_write_complete(void *arg, int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  fs->finished = true;
}
```

文件有数据后，进行文件的读操作

```c++
static void ysxfs_file_read(ysxfs_file_t *file) {
  ysxfs_filesystem_t *fs = file->fs;
  fs->finished = false;
  poller(fs->thread, ysxfs_do_read, file, &fs->finished);
}

static void ysxfs_do_read(void *arg) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  SPDK_NOTICELOG("%s --> enter\n", __func__);
  memset(file->read_buffer, '\0', fs->io_unit_size);
  spdk_blob_io_read(file->blob, fs->channel, file->read_buffer, 0, 1,
                    ysxfs_blob_read_complete, file);
}

static void ysxfs_blob_read_complete(void *arg, int bserrno) {
  ysxfs_file_t *file = (ysxfs_file_t *)arg;
  ysxfs_filesystem_t *fs = file->fs;
  SPDK_NOTICELOG("size: %ld, buffer: %s\n", fs->io_unit_size,
                 file->read_buffer);
  fs->finished = true;
}

static void ysxfs_file_close(ysxfs_file_t *file) {
  if (file->read_buffer) {
    spdk_free(file->read_buffer);
    file->read_buffer = NULL;
  }
  if (file->write_buffer) {
    spdk_free(file->write_buffer);
    file->write_buffer = NULL;
  }
}
```

完成spdk的文件系统初始化、文件操作后，对于上层的Posix接口，还需要提供一层适配层，适配Posix操作文件时使用的文件句柄fd。进行适配层API 的封装开发

```c++
// 封装fd
#define MAX_FD_COUNT 1024
#define DEFAULT_FD_NUM 3

ysxfs_file_t *files[MAX_FD_COUNT] = {0};
static unsigned fd_table[MAX_FD_COUNT / 8] = {0};

static int ysxfs_get_fd(void) {
  int fd = DEFAULT_FD_NUM;
  for (; fd < MAX_FD_COUNT; fd++) {
    if ((fd_table[fd / 8] & (0x1 << (fd % 8))) == 0) {
      fd_table[fd / 8] |= (0x1 << (fd % 8));
      return fd;
    }
  }
  return -1;
}

static void ysxfs_set_fd(int fd) {
  if (fd >= MAX_FD_COUNT)
    return; // errno
  fd_table[fd / 8] &= ~(0x1 << fd % 8);
}

// 文件的创建
static int ysxfs_create(const char *pathname, int flags) {
  if (!fs_instance) {
    ysxfs_filesystem_setup();
  }
  int fd = ysxfs_get_fd();
  ysxfs_file_t *file = calloc(1, sizeof(ysxfs_file_t));
  if (!file) {
    return -1;
  }

  strcpy(file->filename, pathname);

  files[fd] = file;
  file->fs = fs_instance;
  ysxfs_file_create(file);
  return fd;
}

// 文件的写
static ssize_t ysxfs_write(int fd, const void *buf, size_t count) {
  ysxfs_file_t *file = files[fd];
  if (!file)
    return -1;
  memcpy(file->write_buffer, buf, count);
  ysxfs_file_write(file);
  return 0;
}

// 文件的读
static ssize_t ysxfs_read(int fd, void *buf, size_t count) {
  ysxfs_file_t *file = files[fd];
  if (!file)
    return -1;

  ysxfs_file_read(file);
  memcpy(buf, file->read_buffer, count);
  return 0;
}

// 文件的关闭
static int ysxfs_close(int fd) {
  ysxfs_file_t *file = files[fd];
  if (!file)
    return 0;

  ysxfs_file_close(file);
  ysxfs_set_fd(fd);
  free(file);
  files[fd] = NULL;
  return 0;
}
```

最后进行Posxi的函数的hook替换，不修改上次业务，直接底层使用Spdk进行操作。

```c++
#if 1
#define DEBUG_ENABLE 1

#if DEBUG_ENABLE
#define dblog(fmt, ...) printf(fmt, ##__VA_ARGS__)
#else
#define dblog(fmt, ...)
#endif

typedef int (*open_t)(const char *pathname, int flags);
open_t open_f = NULL;

typedef ssize_t (*read_t)(int fd, void *buf, size_t count);
read_t read_f = NULL;

typedef ssize_t (*write_t)(int fd, const void *buf, size_t count);
write_t write_f = NULL;

typedef int (*close_t)(int fd);
close_t close_f = NULL;

int open(const char *pathname, int flags, ...) {
  if (!open_f) {
    open_f = dlsym(RTLD_NEXT, "open");
  }
  dblog("open .. %s\n", pathname);
  return ysxfs_create(pathname, flags);
}

ssize_t read(int fd, void *buf, size_t count) {

  if (!read_f) {
    read_f = dlsym(RTLD_NEXT, "read");
  }
  dblog("read ..\n");
  return ysxfs_read(fd, buf, count);
}

ssize_t write(int fd, const void *buf, size_t count) {
  if (!write_f) {
    write_f = dlsym(RTLD_NEXT, "write");
  }
  dblog("write ..\n");
  return ysxfs_write(fd, buf, count);
}

int close(int fd) {
  if (!close_f) {
    close_f = dlsym(RTLD_NEXT, "close");
  }
  dblog("close ..\n");
  return ysxfs_close(fd);
}
#endif
```

