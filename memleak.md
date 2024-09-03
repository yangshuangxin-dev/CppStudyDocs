# 内存泄漏检测

在C/C++开发中，代码逻辑越来越复杂，就容易发生内存泄漏。内存泄漏的根本原因在于调用了malloc，却没有调用对应的free释放内存，导致应用程序的RSS内存持续增长，可能造成程序、系统崩溃。内存泄漏的检测是非常重要的。

内存泄漏的检测可以适用第三方库如ASAN、valgrind等，也可以自己实现一个简单的内存泄漏检测，直接集成到应用程序中。这样如果内存泄漏的怀疑点，只需要开启一个内存泄漏检测的配置，即可直接进行问题的定位。

定位内存泄漏最重要的俩个信息为，如何判断是否有内存泄漏、在哪一行有内存泄漏，是在哪一行有内存泄漏，下面分别进行讨论。

> malloc申请的最小内存是0x20字节（32字节），malloc申请小于0x20字节时，实际就会申请0x20大小。

## 单文件内存泄漏检测

如果已经定位到可能就是该文件的代码存在内存泄漏，可以直接适用宏替换，对申请和释放的函数进行替换，使用自定义的malloc函数，增加打印、增加文件等方式来定位问题。

```c++
int flag = 1; // 从配置文件获取，可以控制是否开启内存泄漏检测
void *nMalloc(size_t size, const char *filename, int line) {
	void *p = malloc(size);
	if (flag) {
		char buff[128] = {0};
		snprintf(buff, 128, "./mem/%p.mem", p);
		FILE *fp = fopen(buff, "w");
		if (!fp) {
			free(p);
			return NULL;
		}
		// 把内存申请的地址、函数、行号打印到文件中
		fprintf(fp, "[+]%s:%d, addr: %p, size: %ld\n", filename, line, p, size);
		fflush(fp);
		fclose(fp);

	}
	return p;
}

void nFree(void *ptr) {
	if (flag) {
		char buff[128] = {0};
		snprintf(buff, 128, "./mem/%p.mem", ptr);
        // 改内存被释放，删除文件
		if (unlink(buff) < 0) {
			printf("double free: %p", ptr);
			return ;
		}
	}
	
	return free(ptr);
}
// 进行宏替换，可以从文件得到是否内存泄漏，泄漏在哪一行
#define malloc(size) nMalloc(size, __FILE__, __LINE__)
#define free(ptr) nFree(ptr)
```

## 多文件内存泄漏检测

当项目的规模非常大时，不仅仅是自研代码的文件数量非常多，还可能使用到许多的第三方库，内存泄漏的定位就不能使用宏替换了，就使用HOOK系统函数，进行内存泄漏的定位。

> hook系统函数的定位，只有程序带有符号表（编译时增加 -g 编译选项）才能定位到哪一行，否则无法获取具体哪一行，只能判断是否有内存泄漏。

```c++
// 定义需要hook的函数指针
typedef void *(*malloc_t)(size_t size);
malloc_t malloc_f = nullptr;
typedef void (*free_t)(void *ptr);
free_t free_f = nullptr;
// 自定义的malloc函数
void* malloc(size_t size) {
	void* p = NULL;
	p = malloc_f(size);
    // 直接取到上一层调用的函数地址
	void* caller = __builtin_return_address(0);
	char buff[128] = { 0 };
	sprintf(buff, "./mem/%p.mem", p);
	FILE* fp = fopen(buff, "w");
	if (!fp) {
		free(p);
		return nullptr;
	}

	fprintf(fp, "caller:%p, addr: %p, size: %ld\n", caller, p, size);
	fflush(fp);
	fclose(fp);
	return p;
}

// 自定义的释放函数
void free(void* ptr) {
	char buff[128] = { 0 };
	snprintf(buff, 128, "./mem/%p.mem", ptr);
	if (unlink(buff) < 0) {
		printf("double free: %p", ptr);
		return;
	}
	free_f(ptr);
	return;
}
// 对系统和函数进行hook
void init_hook(void) {
	if (!malloc_f) {
		malloc_f = (malloc_t)dlsym(RTLD_NEXT, "malloc");
	}
	if (!free_f) {
		free_f = (free_t)dlsym(RTLD_NEXT, "free");
	}
}

```

通过获取上一层调用的函数地址caller，就可以通过addr2line获取内存泄漏的位置:

```shell
addr2line -f -e 可执行程序 -a caller地址
```

但是在高版本的Linux上，直接使用caller地址进行转换内存泄漏的代码行，可能发生无法解析的问题，就需要对地址进行转换后，才能使用。

```c++
#define _GNU_SOURCE
#include <dlfcn.h>
#include <link.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void* ConvertToELF(void* addr) {
	Dl_info info;
	struct link_map* link;
	dladdr1(addr, &info, (void**)&link, RTLD_DL_LINKMAP);
	return (void*)((size_t)addr - link->l_addr);
}

void* malloc(size_t size) {
	void* p = nullptr;
	p = malloc_f(size);
	void* caller = __builtin_return_address(0);
	char buff[128] = { 0 };
	sprintf(buff, "./mem/%p.mem", p);
	FILE* fp = fopen(buff, "w");
	if (!fp) {
		free(p);
		return NULL;
	}
	fprintf(fp, "ELF: %p, addr: %p, size: %ld\n", ConvertToELF(caller), p, size);
	fflush(fp);
	fclose(fp);
	return p;
}
```

然后同样使用addr2line定位到内存泄漏的位置:

```shell
addr2line -f -e 可执行程序 -a ELF地址
```



## 总结

排查内存泄漏的问题，在单文件可以使用宏替换，可以定位到哪一行有内存泄漏；在多文件时需要hook系统函数，能判断是否有内存泄漏，但是不一定可以知道哪一行有内存泄漏，需要通过地址查找符号表获取内存泄漏的位置，就要求应用程序编译的时候有-g的编译选项，如果是第三方库，同样也需要符号表才能定位。

