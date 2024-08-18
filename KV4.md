# NOSQL （KV存储）增加哈希表

在NOSQL（KV存储）中增加新的存储结构哈希表，进行快速的数据查找和访问。

## 存储字符串的哈希表

1. 定义哈希表结构

    ```c++
    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    #include <pthread.h>
    #define MAX_KEY_LEN	128
    #define MAX_VALUE_LEN	512
    #define MAX_TABLE_SIZE	102400
    // 定义哈希表节点
    typedef struct hashnode_s {
    	char *key;
    	char *value;
    	struct hashnode_s *next;
    } hashnode_t;
    
    // 定义哈希表
    typedef struct hashtable_s {
    	hashnode_t **nodes;
    	int max_slots;
    	int count;
    } hashtable_t;
    
    static hashtable_t Hash;
    ```

2. 定义哈希函数

    ```c++
    static int _hash(char *key, int size) {
    	if (!key) {
            return -1;
        }
    	int sum = 0;
    	int i = 0;
    	while (key[i] != 0) {
    		sum += key[i];
    		i ++;
    	}
    	return sum % size;
    }
    ```

3. 初始化和销毁哈希表

    ```c++
    // 创建哈希表
    int init_hashtable(hashtable_t *hash) {
    	if (!hash) {
            return -1;
        }
    	hash->nodes = (hashnode_t**)kvstore_malloc(sizeof(hashnode_t*) * MAX_TABLE_SIZE);
    	if (!hash->nodes) {
            return -1;
        }
    	hash->max_slots = MAX_TABLE_SIZE;
    	hash->count = 0; 
    	return 0;
    }
    // 创建哈希表的节点
    hashnode_t *_create_node(char *key, char *value) {
    	hashnode_t *node = (hashnode_t*)kvstore_malloc(sizeof(hashnode_t));
    	if (!node) {
            return NULL;
        }
    	node->key = kvstore_malloc(strlen(key) + 1);
    	if (!node->key) {
    		kvstore_free(node);
    		return NULL;
    	}
    	strcpy(node->key, key);
    	node->value = kvstore_malloc(strlen(value) + 1);
    	if (!node->value) {
    		kvstore_free(node->key);
    		kvstore_free(node);
    		return NULL;
    	}
    	strcpy(node->value, value);
    	node->next = NULL;
    	return node;
    }
    // 销毁哈希表的节点和哈希表
    void dest_hashtable(hashtable_t *hash) {
    	if (!hash) {
            return;
        }
    	int i = 0;
    	for (i = 0;i < hash->max_slots;i ++) {
    		hashnode_t *node = hash->nodes[i];
    		while (node != NULL) { // error
    			hashnode_t *tmp = node;
    			node = node->next;
    			hash->nodes[i] = node;
    			kvstore_free(tmp);		
    		}
    	}
    	kvstore_free(hash->nodes);	
    }
    ```

4. 哈希表中插入元素

    ```c++
    int put_kv_hashtable(hashtable_t *hash, char *key, char *value) {
    	if (!hash || !key || !value) {
            return -1;
        }
    	int idx = _hash(key, MAX_TABLE_SIZE);
    	hashnode_t *node = hash->nodes[idx];
    	while (node != nullptr) {
    		if (strcmp(node->key, key) == 0) { // exist
    			return 1;
    		}
    		node = node->next;
    	}
    	hashnode_t *new_node = _create_node(key, value);
    	new_node->next = hash->nodes[idx];
    	hash->nodes[idx] = new_node;
    	hash->count ++;
    	return 0;
    }
    ```

5. 哈希表中获取元素

    ```c++
    char * get_kv_hashtable(hashtable_t *hash, char *key) {
    	if (!hash || !key) {
            return NULL;
        }
    	int idx = _hash(key, MAX_TABLE_SIZE);
    	hashnode_t *node = hash->nodes[idx];
    	while (node != NULL) {
    		if (strcmp(node->key, key) == 0) {
    			return node->value;
    		}
    		node = node->next;
    	}
    	return NULL;
    }
    // 获取哈希表中元素的数量
    int count_kv_hashtable(hashtable_t *hash) {
    	return hash->count;
    }
    ```

6. 哈希表中删除元素

    ```c++
    int delete_kv_hashtable(hashtable_t *hash, char *key) {
    	if (!hash || !key) {
            return -2;
        }
    	int idx = _hash(key, MAX_TABLE_SIZE);
    	hashnode_t *head = hash->nodes[idx];
    	if (head == NULL) {
            return -1;
        }
    	// head node
    	if (strcmp(head->key, key) == 0) {
    		hashnode_t *tmp = head->next;
    		hash->nodes[idx] = tmp;
    		if (head->key) {
    			kvstore_free(head->key);
    		}
    		if (head->value) {
    			kvstore_free(head->value);
    		}
    		kvstore_free(head);
    		hash->count --;
    		return 0;
    	}
    	// 查找删除的节点
    	hashnode_t *cur = head;
    	while (cur->next != NULL) {
    		if (strcmp(cur->next->key, key) == 0){
                break;
            }
    		cur = cur->next;
    	}
        // 找不到节点
    	if (cur->next == NULL) {	
    		return -1;
    	}
    
    	hashnode_t *tmp = cur->next;
    	cur->next = tmp->next;
    	// 删除节点
    	if (tmp->key) {
    		kvstore_free(tmp->key);
    	}
    	if (tmp->value) {
    		kvstore_free(tmp->value);
    	}
    	kvstore_free(tmp);
    	hash->count--;
    	return 0;
    }
    ```

7. 哈希表的修改元素

    ```c++
    int mod_kv_hashtable(hashtable_t *hash, char *key, char *value) {
    	if (!hash || !key || !value) {
            return -1;
        }
    	int idx = _hash(key, MAX_TABLE_SIZE);
    	hashnode_t *node = hash->nodes[idx];
    	while (node != NULL) {
    		if (strcmp(node->key, key) == 0) {
    			kvstore_free(node->value);
    			node->value = kvstore_malloc(strlen(value) + 1);
    			if (node->value) {
    				strcpy(node->value, value);
    				return 0;
    			} else {
                    return -1;
                }
    		}
    		node = node->next;
    	}
    	return -1;
    }
    ```

    

## KV存储的接口层

接口层是对存储哈希表原生接口进行封装，使其可以对接KV存储的接口。

1. 哈希表的创建和销毁

    ```c++
    int kvstore_hash_create(hashtable_t *hash) {
    	return init_hashtable(hash);
    }
    
    void kvstore_hash_destory(hashtable_t *hash) {
    	return dest_hashtable(hash);
    }
    ```

2. 哈希表的增删改查

    ```c++
    int kvs_hash_set(hashtable_t *hash, char *key, char *value) {
    	return put_kv_hashtable(hash, key, value);
    }
    
    char *kvs_hash_get(hashtable_t *hash, char *key) {
    	return get_kv_hashtable(hash, key);
    }
    
    int kvs_hash_delete(hashtable_t *hash, char *key) {
    	return delete_kv_hashtable(hash, key);
    }
    
    int kvs_hash_modify(hashtable_t *hash, char *key) {
    	return mod_kv_hashtable(hash, key);
    }
    ```

## KV存储的适配层和核心实现

1. 哈希表KV存储的适配层

    ```c++
    int kvstore_hash_set(char *key, char *value) {
    	return kvs_hash_set(&Hash, key, value);
    }
    
    char *kvstore_hash_get(char *key) {
    	return kvs_hash_get(&Hash, key);
    }
    
    int kvstore_hash_delete(char *key) {
    	return kvs_hash_delete(&Hash, key);
    }
    
    int kvstore_hash_modify(char *key, char *value) {
    	return kvs_hash_modify(&Hash, key, value);
    }
    
    int kvstore_hash_count(void) {
    	return kvs_hash_count(&Hash);
    }
    ```

2. 增加命令字

    ```c++
    const char *commands[] = {
    	"SET", "GET", "DEL", "MOD", "COUNT",	// 数组实现
    	"RSET", "RGET", "RDEL", "RMOD", "RCOUNT", // 红黑树实现
    	"HSET", "HGET", "HDEL", "HMOD", "HCOUNT", // 哈希表实现
    };
    
    enum {
       // 数组实现
    	KVS_CMD_START = 0,
    	KVS_CMD_SET = KVS_CMD_START,
    	KVS_CMD_GET,
    	KVS_CMD_DEL,
    	KVS_CMD_MOD,
    	KVS_CMD_COUNT,
    	// 红黑树实现
    	KVS_CMD_RSET,
    	KVS_CMD_RGET,
    	KVS_CMD_RDEL,
    	KVS_CMD_RMOD,
    	KVS_CMD_RCOUNT,
    	// 哈希表实现
    	KVS_CMD_HSET,
    	KVS_CMD_HGET,
    	KVS_CMD_HDEL,
    	KVS_CMD_HMOD,
    	KVS_CMD_HCOUNT,
    	
    	KVS_CMD_SIZE,
    };
    ```

3. 核心逻辑的实现

    ```c++
    		case KVS_CMD_HSET: {
    			int res = kvstore_hash_set(key, value);
    			if (!res) {
    				snprintf(msg, BUFFER_LENGTH, "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "FAILED");
    			}
    			break;
    		}
    		case KVS_CMD_HGET: {
    			char *val = kvstore_hash_get(key);
    			if (val) {
    				snprintf(msg, BUFFER_LENGTH, "%s", val);
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			break;
    		}
    		case KVS_CMD_HDEL: {
    			int res = kvstore_hash_delete(key);
    			if (res < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else if (res == 0) {
    				snprintf(msg, BUFFER_LENGTH, "%s", "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			break;
    		}
    		case KVS_CMD_HMOD: {
    			int res = kvstore_hash_modify(key, value);
    			if (res < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else if (res == 0) {
    				snprintf(msg, BUFFER_LENGTH, "%s", "SUCCESS");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "NO EXIST");
    			}
    			break;
    		}
    		case KVS_CMD_HCOUNT: {
    			int count = kvstore_hash_count();
    			if (count < 0) {  // server
    				snprintf(msg, BUFFER_LENGTH, "%s", "ERROR");
    			} else {
    				snprintf(msg, BUFFER_LENGTH, "%d", count);
    			}
    			break;
    		}
    ```

## KV存储哈希表的测试用例

```c++
void hash_testcase(int connfd) {
	test_case(connfd, "HSET Name YangShuangXin", "SUCCESS", "HSETCase");
	test_case(connfd, "HGET Name", "YangShuangXin", "HGETCase");
	test_case(connfd, "HMOD Name yangshuangxin", "SUCCESS", "HMODCase");
	test_case(connfd, "HGET Name", "yangshuangxin", "HGETCase");
	test_case(connfd, "HDEL Name", "SUCCESS", "HDELCase");
	test_case(connfd, "HGET Name", "NO EXIST", "HGETCase");
}

void hash_testcase_lot(int connfd) {
	int count = 100000; // 10w
	int i = 0;
	while (i ++ < count) {
		hash_testcase(connfd);
	}
}

void hash_testcase_node(int connfd) {
	int count = 50000; // 5w
	int i = 0;
    // 增加节点
	for (i = 0;i < count;i ++) {
		char cmd[128] = {0};
		snprintf(cmd, 128, "HSET Name%d Yangshuangxin%d", i, i);
		test_case(connfd, cmd, "SUCCESS", "SETCase");
		char result[128] = {0};
		sprintf(result, "%d", i+1);
		test_case(connfd, "HCOUNT", result, "HCOUNT");
	}
	// 删除节点
	for (i = 0;i < count;i ++) {
		char cmd[128] = {0};
		snprintf(cmd, 128, "HDEL Name%d King%d", i, i);
		test_case(connfd, cmd, "SUCCESS", "DELCase");
		char result[128] = {0};
		sprintf(result, "%d", count - (i+1));
		test_case(connfd, "HCOUNT", result, "RCOUNT");
	}
}
```

## KV存储不同语言的客户端

### Golang

```go
package main
import (
	"fmt"
	"net"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:9096")
	if err != nil {
		fmt.Println("connect failed: ", err)
		os.Exit(1)
	}
	defer conn.Close()
    
	message := "SET NameGolang Yangshuangxin"
	_, err = conn.Write([]byte(message))
	if err != nil {
		fmt.Println("send failed: ", err)
		os.Exit(1)
	}

	fmt.Printf("send msg: %s\n", message)

	buffer := make([]byte, 1024)
	length, err := conn.Read(buffer)
	if err != nil {
		fmt.Println("recv failed: ", err)
		os.Exit(1)
	}
	response := string(buffer[:length])
	fmt.Printf("recv msg: %s\n", response)
}
```

### JAVA

```java
import java.io.*;
import java.net.*;

public class javakvstore {
    public static void main(String[] args) {
        String serverAddress = "127.0.0.1"; 
        int serverPort = 9096;  
        try {
            Socket socket = new Socket(serverAddress, serverPort);
            OutputStream outputStream = socket.getOutputStream();
            InputStream inputStream = socket.getInputStream();

            String message = "SET NameJAVA Yangshuangxin";
            outputStream.write(message.getBytes());

            byte[] buffer = new byte[1024];
            int bytesRead = inputStream.read(buffer);
            if (bytesRead > 0) {
                String response = new String(buffer, 0, bytesRead);
                System.out.println("recv: " + response);
            }

			inputStream.close();
			outputStream.close();
			socket.close();

		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

### Nodejs

```javascript
const net = require('net');

const client = net.createConnection({ port: 9096, host: '127.0.0.1' }, () => {
  console.log('connect kvstore');

  client.write('SET NameJS Yangshuangxin');
});

client.on('data', (data) => {
  console.log(`recv：${data.toString()}`);
  client.end();
});

client.on('error', (err) => {
  console.error('connect failed：', err);
});

client.on('close', () => {
  console.log('close connection');
});
```

### Python

```python
import socket

def main():
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    server_address = ('127.0.0.1', 9096)  

    try:
        client_socket.connect(server_address)

        message = "SET NamePython Yangshuangxin"
        client_socket.sendall(message.encode())

        response = client_socket.recv(1024)
        
        print("recv:", response.decode())
    
    except Exception as e:
        print("发生异常:", str(e))
    
    finally:
        client_socket.close()

if __name__ == '__main__':
    main()

```

### Rust

```rust
use std::io::{self, Read, Write};
use std::net::TcpStream;

fn main() -> io::Result<()> {
    let server_address = "127.0.0.1:9096";

    let mut stream = TcpStream::connect(server_address)?;

    stream.write_all(b"SET NameRust Yangshuangxin")?;

    let mut buffer = [0; 1024];
    let bytes_read = stream.read(&mut buffer)?;
    println!("Received: {}", String::from_utf8_lossy(&buffer[..bytes_read]));

    Ok(())
}
```

