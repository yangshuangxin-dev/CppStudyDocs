#	NOSQL （KV存储）增加红黑树存储

在NOSQL（KV存储）中，仅仅使用数组是不满足快速访问和存储的需求，为了使KV存储的速度更快，引入红黑树作为后端的存储数据结构。

## 存储字符串的红黑树

1. 定义字符串类型的红黑树

    ```c++
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    
    #define RED				1
    #define BLACK 			2
    #define MAX_KEY_LEN			256
    #define MAX_VALUE_LEN		1024
    // key 类型，字符串
    typedef char* KEY_TYPE;
    // 红黑树节点
    typedef struct {
    	unsigned char color;
    	struct _rbtree_node *right;
    	struct _rbtree_node *left;
    	struct _rbtree_node *parent;
    
    	KEY_TYPE key;
    	void *value;
    } rbtreeNode;
    // 红黑树
    typedef struct {
    	rbtree_node *root;
    	rbtree_node *nil;
    	
    	int count;
    } rbtree;
    ```

    

2. 红黑树基本操作：最小节点、最大节点、后继结点、左旋、右旋

    ```c++
    // 返回最小节点
    rbtreeNode *rbtreeMini(rbtree *T, rbtreeNode *x) {
    	while (x->left != T->nil) {
    		x = x->left;
    	}
    	return x;
    }
    
    // 返回最大节点
    rbtreeNode *rbtreeMaxi(rbtree *T, rbtreeNode *x) {
    	while (x->right != T->nil) {
    		x = x->right;
    	}
    	return x;
    }
    
    // 后继结点
    rbtreeNode *rbtreeSuccessor(rbtree *T, rbtreeNode *x) {
    	rbtreeNode *y = x->parent;
    
    	if (x->right != T->nil) {
    		return rbtreeMini(T, x->right);
    	}
    
    	while ((y != T->nil) && (x == y->right)) {
    		x = y;
    		y = y->parent;
    	}
    	return y;
    }
    // 红黑树左旋
    void rbtreeLeftRotate(rbtree *T, rbtreeNode *x) {
    	rbtree_node *y = x->right;  // x  --> y  ,  y --> x,   right --> left,  left --> right
    	x->right = y->left;
    	if (y->left != T->nil) {
    		y->left->parent = x;
    	}
    	y->parent = x->parent;
    	if (x->parent == T->nil) {
    		T->root = y;
    	} else if (x == x->parent->left) {
    		x->parent->left = y;
    	} else {
    		x->parent->right = y;
    	}
    	y->left = x;
    	x->parent = y;
    }
    
    // 红黑树右旋
    void rbtreeRightRotate(rbtree *T, rbtreeNode *y) {
    	rbtreeNode *x = y->left;
    	y->left = x->right;
    	if (x->right != T->nil) {
    		x->right->parent = y;
    	}
    	x->parent = y->parent;
    	if (y->parent == T->nil) {
    		T->root = x;
    	} else if (y == y->parent->right) {
    		y->parent->right = x;
    	} else {
    		y->parent->left = x;
    	}
    	x->right = y;
    	y->parent = x;
    }
    ```

3. 红黑树插入和插入后进行红黑树重新平衡

    ```c++
    // 红黑树进行插入节点后进行重新平衡为红黑树
    void rbtreeInsertFixup(rbtree *T, rbtreeNode *z) {
    	while (z->parent->color == RED) { //z ---> RED
    		if (z->parent == z->parent->parent->left) {
    			rbtreeNode *y = z->parent->parent->right;
    			if (y->color == RED) {
    				z->parent->color = BLACK;
    				y->color = BLACK;
    				z->parent->parent->color = RED;
    				z = z->parent->parent; //z --> RED
    			} else {
    				if (z == z->parent->right) {
    					z = z->parent;
    					rbtree_left_rotate(T, z);
    				}
    				z->parent->color = BLACK;
    				z->parent->parent->color = RED;
    				rbtreeRightRotate(T, z->parent->parent);
    			}
    		}else {
    			rbtreeNode *y = z->parent->parent->left;
    			if (y->color == RED) {
    				z->parent->color = BLACK;
    				y->color = BLACK;
    				z->parent->parent->color = RED;
    
    				z = z->parent->parent; //z --> RED
    			} else {
    				if (z == z->parent->left) {
    					z = z->parent;
    					rbtree_right_rotate(T, z);
    				}
    				z->parent->color = BLACK;
    				z->parent->parent->color = RED;
    				rbtreeLeftRotate(T, z->parent->parent);
    			}
    		}
    	}
    	T->root->color = BLACK;
    }
    
    // 红黑树插入节点（字符串节点）
    void rbtreeInsert(rbtree *T, rbtreeNode *z) {
    	rbtreeNode *y = T->nil;
    	rbtreeNode *x = T->root;
    
    	while (x != T->nil) {
    		y = x;
    // key是字符串，字符串比较使用strcmp
    		if (strcmp(z->key, x->key) < 0) {
    			x = x->left;
    		} else if (strcmp(z->key, x->key) > 0) {
    			x = x->right;
    		} else {
    			return ;
    		}
    	}
    
    	z->parent = y;
    	if (y == T->nil) {
    		T->root = z;
    // key是字符串，字符串比较使用strcmp
    	} else if (strcmp(z->key, y->key) < 0) {
    		y->left = z;
    	} else {
    		y->right = z;
    	}
    
    	z->left = T->nil;
    	z->right = T->nil;
    	z->color = RED;
    	// 插入节点后进行重新平衡修复
    	rbtreeInsertFixup(T, z);
    }
    ```

4. 红黑树删除和进行修复

    ```c++
    // 红黑树删除后进行修复
    void rbtreeDeleteFixup(rbtree *T, rbtreeNode *x) {
    	while ((x != T->root) && (x->color == BLACK)) {
    		if (x == x->parent->left) {
    			rbtreeNode *w= x->parent->right;
    			if (w->color == RED) {
    				w->color = BLACK;
    				x->parent->color = RED;
    				rbtree_left_rotate(T, x->parent);
    				w = x->parent->right;
    			}
    			if ((w->left->color == BLACK) && (w->right->color == BLACK)) {
    				w->color = RED;
    				x = x->parent;
    			} else {
    				if (w->right->color == BLACK) {
    					w->left->color = BLACK;
    					w->color = RED;
    					rbtree_right_rotate(T, w);
    					w = x->parent->right;
    				}
    				w->color = x->parent->color;
    				x->parent->color = BLACK;
    				w->right->color = BLACK;
    				rbtreeLeftRotate(T, x->parent);
    				x = T->root;
    			}
    		} else {
    			rbtreeNode *w = x->parent->left;
    			if (w->color == RED) {
    				w->color = BLACK;
    				x->parent->color = RED;
    				rbtreeRightRotate(T, x->parent);
    				w = x->parent->left;
    			}
    
    			if ((w->left->color == BLACK) && (w->right->color == BLACK)) {
    				w->color = RED;
    				x = x->parent;
    			} else {
    				if (w->left->color == BLACK) {
    					w->right->color = BLACK;
    					w->color = RED;
    					rbtree_left_rotate(T, w);
    					w = x->parent->left;
    				}
    				w->color = x->parent->color;
    				x->parent->color = BLACK;
    				w->left->color = BLACK;
    				rbtreeRightRotate(T, x->parent);
    				x = T->root;
    			}
    		}
    	}
    	x->color = BLACK;
    }
    // 红黑树删除
    rbtreeNode *rbtreeDelete(rbtree *T, rbtreeNode *z) {
    	rbtreeNode *y = T->nil;
    	rbtreeNode *x = T->nil;
    	if ((z->left == T->nil) || (z->right == T->nil)) {
    		y = z;
    	} else {
    		y = rbtreeSuccessor(T, z);
    	}
    	if (y->left != T->nil) {
    		x = y->left;
    	} else if (y->right != T->nil) {
    		x = y->right;
    	}
    
    	x->parent = y->parent;
    	if (y->parent == T->nil) {
    		T->root = x;
    	} else if (y == y->parent->left) {
    		y->parent->left = x;
    	} else {
    		y->parent->right = x;
    	}
    
    	if (y != z) {
    // 字符串指针交换（key和value）
    		void *tmp = z->key;
    		z->key = y->key;
    		y->key = tmp;
    
    		tmp = z->value;
    		z->value = y->value;
    		y->value = tmp;
    	}
    
    	if (y->color == BLACK) {
    		rbtreeDeleteFixup(T, x);
    	}
    	return y;
    }
    ```

5. 红黑树的查找和遍历

    ```c++
    // 红黑树的查找
    rbtreeNode *rbtreeSearch(rbtree *T, KEY_TYPE key) {
    	rbtreeNode *node = T->root;
    	while (node != T->nil) {
    // 字符串类型的key，使用strcmp 比较
    		if (strcmp(key, node->key) < 0) {
    			node = node->left;
    		} else if (strcmp(key, node->key) > 0) {
    			node = node->right;
    		} else {
    			return node;
    		}
    	}
    	return T->nil;
    }
    // 字符串的遍历
    void rbtreeTraversal(rbtree *T, rbtreeNode *node) {
    	if (node != T->nil) {
    		rbtreeTraversal(T, node->left);
    		printf("key:%s, color:%d\n", node->key, node->color);
    		rbtreeTraversal(T, node->right);
    	}
    }
    
    ```

## KV存储的红黑树适配层

有了红黑树的数据结构，还需要增加KV存储的适配层，连接KV存储和底层的红黑树数据结构。

1. 红黑树的创建和销毁

    ```c++
    // 创建红黑树
    int kvstoreRbtreeCreate(rbtree *tree) {
    	if (!tree) {
            return -1;
        }
    	memset(tree, 0, sizeof(rbtree));
    	tree->nil = (rbtree_node*)malloc(sizeof(rbtree_node));
    	tree->nil->key = malloc(1);
    	*(tree->nil->key) = '\0';
    	tree->nil->color = BLACK;
    	tree->root = tree->nil;
    	return 0;
    }
    // 销毁红黑树
    void kvstoreRbtreeDestory(rbtree *tree) {
    	if (!tree) {
            return;
        }
    	if (tree->nil->key) {
            kvstoreFree(tree->nil->key);
        }
    	rbtreeNode *node = tree->root;
    	while (node != tree->nil) {
    		node = rbtreeMini(tree, tree->root);
    		if (node == tree->nil) {
    			break;
    		}
    		node = rbtreeDelete(tree, node);
    		if (!node) {
    			kvstoreFree(node->key);
    			kvstoreFree(node->value);
    			kvstoreFree(node);
    		}
    	}
    }
    ```

2. KV存储存入KV键值对

    ```c++
    int kvsRbtreeSet(rbtree *tree, char *key, char *value) {
    	rbtreeNode *node  = (rbtreeNode*)malloc(sizeof(rbtreeNode));
    	if (!node) {
            return -1;
        }
    	node->key = kvstoreMalloc(strlen(key) + 1);
    	if (node->key == nullptr) {
    		kvstore_free(node);
    		return -1;
    	}
    	memset(node->key, 0, strlen(key) + 1);
    	strcpy(node->key, key);
    	node->value = kvstoreMalloc(strlen(value) + 1);
    	if (node->value == NULL) {
    		kvstoreFree(node->key);
    		kvstoreFree(node);
    		return -1;
    	}
    	memset(node->value, 0, strlen(value) + 1);
    	strcpy((char *)node->value, value);
    	rbtreeInsert(tree, node);
    	tree->count ++;
    	return 0;
    }
    ```

3. KV存储获取VALUE

    ```c++
    char* kvsRbtreeGet(rbtree *tree, char *key) {
    	rbtreeNode *node = rbtreeSearch(tree, key);
    	if (node == tree->nil) {
    		return nullptr;
    	}
    	return node->value;
    }
    ```

4. KV存储删除节点

    ```c++
    int kvsRbtreeDelete(rbtree *tree, char *key) {
    	rbtreeNode *node = rbtreeSearch(tree, key);
    	if (node == tree->nil) {
    		return -1;
    	}
    	rbtreeNode *cur = rbtreeDelete(tree, node);
    	if (!cur) {
    		kvstoreFree(cur->key);
    		kvstoreFree(cur->value);
    		kvstoreFree(cur);
    	}
    	tree->count --;
    	return 0;
    }
    ```

5. KV存储修改节点

    ```c++
    int kvsRbtreeModify(rbtree *tree, char *key, char *value) {
    	rbtreeNode *node = rbtreeSearch(tree, key);
    	if (node == tree->nil) {
    		return -1;
    	}
    	char *tmp = node->value;
    	kvstoreFree(tmp);
    	node->value = kvstoreMalloc(strlen(value) + 1);
    	if (node->value == NULL) {
    		return -1;
    	}
    	strcpy(node->value, value);
    	return 0;
    }
    ```

6. KV存储统计存储数量

    ```c++
    int kvs_rbtree_count(rbtree *tree) {
    	return tree->count;
    }
    ```

## KV存储的红黑树操作

1. KV存储初始化红黑树和增加命令字

    ```c++
    const char *commands[] = {
        // 数组操作
    	"SET", "GET", "DEL", "MOD", "COUNT",
        // 红黑树操作
    	"RSET", "RGET", "RDEL", "RMOD", "RCOUNT",
    };
    enum {
        // 数组操作
    	KVS_CMD_START = 0,
    	KVS_CMD_SET = KVS_CMD_START,
    	KVS_CMD_GET,
    	KVS_CMD_DEL,
    	KVS_CMD_MOD,
    	KVS_CMD_COUNT,
    	// 红黑树操作
    	KVS_CMD_RSET,
    	KVS_CMD_RGET,
    	KVS_CMD_RDEL,
    	KVS_CMD_RMOD,
    	KVS_CMD_RCOUNT,
    	KVS_CMD_SIZE,
    };
    // 创建红黑树
    kvstoreRbtreeCreate(&Tree);
    // 封装接口
    int kvstoreRbtreeSet(char *key, char *value) {
    	return kvsRbtreeSet(&Tree, key, value);
    }
    char* kvstoreRbtreeGet(char *key) {
    	return kvsRbtreeGet(&Tree, key);
    }
    int kvstoreRbtreeDelete(char *key) {
    	return kvsRbtreeDelete(&Tree, key);
    }
    int kvstoreRbtreeModify(char *key, char *value) {
    	return kvsRbtreeModify(&Tree, key, value);
    }
    int kvstoreRbtreeCount(void) {
    	return kvsRbtreeCount(&Tree);
    }
    ```

2. KV存储的红黑树操作

    ```c++
    		// 红黑树增加KV
    		case KVS_CMD_RSET: {
    			int res = kvstoreRbtreeSet(key, value);
    			if (!res) {
    				snprintf(msg, BUFFER_LENGTH, "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "FAILED");
    			}
    			break;
    		}
    		// 红黑树查找KV
    		case KVS_CMD_RGET: {
    			char *val = kvstoreRbtreeGet(key);
    			if (val) {
    				snprintf(msg, BUFFER_LENGTH, "%s", val);
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			break;
    		}
    		// 红黑树删除KV
    		case KVS_CMD_RDEL: {
    			int res = kvstoreRbtreeDelete(key);
    			if (res < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else if (res == 0) {
    				snprintf(msg, BUFFER_LENGTH, "%s", "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			break;
    		}
    		// 红黑树修改KV
    		case KVS_CMD_RMOD: {
    			int res = kvstoreRbtreeModify(key, value);
    			if (res < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else if (res == 0) {
    				snprintf(msg, BUFFER_LENGTH, "%s", "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			
    			break;
    		}
    		// 统计数量
    		case KVS_CMD_RCOUNT: {
    			int count = kvstoreRbtreeCount();
    			if (count < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "%d", count);
    			}
    			break;
    		}
    ```

    ## 红黑树测试用例

    ```c++
    void rbtreeTestcase(int connfd) {
    	testCase(connfd, "RSET Name YangShuangXin", "SUCCESS", "SETCase");
    	testCase(connfd, "RGET Name", "YangShuangXin", "GETCase");
    	testCase(connfd, "RMOD Name yangshuangxin", "SUCCESS", "MODCase");
    	testCase(connfd, "RGET Name", "yangshuangxin", "GETCase");
    	testCase(connfd, "RDEL Name", "SUCCESS", "DELCase");
    	testCase(connfd, "RGET Name", "NO EXIST", "GETCase");
    }
    // 执行10w次
    void rbtreeTestcase(int connfd) {
    	int count = 100000;
    	int i = 0;
    	while (i ++ < count) {
    		rbtreeTestcase(connfd);
    	}
    }
    // 插入5w个红黑树节点
    void rbtreeTestcaseNode(int connfd) {
    	int count = 50000;
    	int i = 0;
        // 插入节点
    	for (i = 0;i < count;i ++) {
    		char cmd[128] = {0};
    		snprintf(cmd, 128, "RSET Name%d YangShuangXin%d", i, i);
    		testCase(connfd, cmd, "SUCCESS", "SETCase");
    		char result[128] = {0};
    		sprintf(result, "%d", i+1);
    		testCase(connfd, "RCOUNT", result, "RCOUNT");
    	}
    	// 删除节点
    	for (i = 0;i < count;i ++) {
    		char cmd[128] = {0};
    		snprintf(cmd, 128, "RDEL Name%d YangShuangXin%d", i, i);
    		testCase(connfd, cmd, "SUCCESS", "DELCase");
    		char result[128] = {0};
    		sprintf(result, "%d", count - (i+1));
    		testCase(connfd, "RCOUNT", result, "RCOUNT");
    	}
    }
    ```

    