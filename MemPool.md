# 内存池

在应用开发中，内存池也常常需要被使用，作为应用程序的一个基础组件。最常用的就是需要分配一些固定大小的块，这些块可以从内存池中获取，避免系统的频繁分配造成的长期运行形成的内存碎片，比如服务端的应用程序可以为每一个客户端的连接分配一个内存池，该客户端的所有的数据处理都从自己的内存池中分配内存进行处理，连接建立的时候分配，连接断开的时候销毁。

> 开源内存池有jemalloc和tcmalloc，可以不修改代码直接使用内存池。

## 固定内存块的内存池

该内存池的应用场景为每一次都需要相同的内存，分别的时间和释放的时间可以任意，例如内存数据库中。

1. 定义内存池

    ```c++
    typedef struct mempool_s {
        int32_t block_size;	// 固定块的大小
        int32_t free_count;	// 空闲块的数量
        char *free_ptr;		//指向下一个可分配的块
        char *mem;
    } mempool_t;
    
    ```

2. 初始化内存池

    ```c++
    #define MEM_PAGE_SIZE		4096 // 4K，页大小
    int mpInit(mempool_t *m, int size) {
    
    	if (!m) {
            return -1;
        }
        
    	if (size < 16) {
            size = 16;
        }
    	m->block_size = size;
    
    	m->mem = (char *)malloc(MEM_PAGE_SIZE);
    	if (!m->mem) {
            return -1;
        }
    	m->free_ptr = m->mem;
    	m->free_count = MEM_PAGE_SIZE / size;
    
    	int i = 0;
    	char *ptr = m->free_ptr;
    	for (i = 0;i < m->free_count;i ++) {
    		*(char **)ptr = ptr + size; // 指向下一个块
    		ptr += size;
    	}
    	*(char **)ptr = nullptr;
    	return 0;
    }
    
    ```

3. 销毁内存池

    ```c++
    void mpDest(mempool_t *m) {
    	if (!m || !m->mem) {
            return;
        }
    	free(m->mem);
    }
    ```

4. 从内存池分配内存块

    ```c++
    void *mpAlloc(mempool_t *m) {
    	if (!m || m->free_count == 0) 
        {
            return NULL;
        }
    	void *ptr = m->free_ptr;
    	m->free_ptr = *(char **)ptr;
    	m->free_count --;
    	return ptr;
    }
    ```

5. 向内存池释放块

    ```c++
    void mpFree(mempool_t *m, void *ptr) {
        // 头插法
    	*(char **)ptr = m->free_ptr;
    	m->free_ptr = (char *)ptr;
    	m->free_count ++;
    }
    ```

## 大小不固定销毁但时间固定的内存池

该内存池的应用场景为每一次分配的大小是不固定的，但是有一个统一的释放时间，例如针对客户端的连接，连接建立的时候分配，连接断开的时候销毁。该内存池由许多页组成，每一个页组成一个链表。

1. 定义内存池的节点页

    ```c++
    typedef struct mp_node_s {
    	char *free_ptr;
    	char *end;
    	struct mp_node_s *next;
    } mp_node_t;
    
    ```

2. 定义内存池

    ```c++
    typedef struct mp_pool_s {
    	struct mp_node_s *first;
    	struct mp_node_s *current;
    	int max; // page size
    } mp_pool_t;
    ```

3. 初始化内存池

    ```c++
    #define MEM_PAGE_SIZE		4096
    int32_t mpInit(mp_pool_t *m, int32_t size = MEM_PAGE_SIZE) {
    	if (!m) return {
            -1;
        }
    	void *addr = malloc(size);
    	mp_node_t *node = (mp_node_t*)addr;
    	// 该页可分配的内存
    	node->free_ptr = (char*)addr + sizeof(mp_node_t);
    	node->end = (char*)addr + size;
    	node->next = NULL;
    	// 内存池指向第一个页
    	m->first = node;
    	m->current = node;
    	m->max = size;
    	return 0;
    }
    ```

4. 销毁内存池

    ```c++
    void mpDest(mp_pool_t *m) {
    	if (!m) {
            return ;
        }
    	while (m->first != nullptr) {
    		void *addr = m->first;
    		mp_node_t *node = (mp_node_t*)addr;
    		m->first = node->next;
    		// 释放页	
    		free(addr);
    	}
    	return ;
    }
    ```

5. 从内存池分配不固定大小的内存

    ```c++
    void *mpMlloc(mp_pool_t *m, int size) { //size < (4096-sizeof(mp_node_t))
    	void *addr = m->first;
    	mp_node_t *node = (mp_node_t*)addr;
    	do {
    		if (size <= (node->end - node->free_ptr)) { // 遍历链表，找到可分配的页
    			char *ptr = node->free_ptr;
    			node->free_ptr += size;
    			return ptr;
    		}
    		node = node->next;
    	} while (node);
    
    	// 无可分配的页，新建页
    	addr = malloc(m->max); // 4096
    	node = (mp_node_t*)addr;
    	node->free_ptr = (char*)addr + sizeof(mp_node_t);
    	node->end = (char*)addr + m->max;
    	// 插入内存池（头插法）
    	node->next = m->current;
    	m->current = node;
    	// 分配内存
    	char *ptr = node->free_ptr;
    	node->free_ptr += size;
    	return ptr;
    }
    
    ```

    