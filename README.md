### 功能性测试

**发送端**

```C++
#include <iostream>
#include <fstream>
#include <string>
#include <cstring>
#include <openssl/md5.h>
#include <arpa/inet.h>

using namespace std;

// 计算MD5值
string md5sum(char *data, int len) {
    MD5_CTX md5ctx;
    MD5_Init(&md5ctx);
    MD5_Update(&md5ctx, data, len);
    unsigned char md5[MD5_DIGEST_LENGTH];
    MD5_Final(md5, &md5ctx);
    char result[MD5_DIGEST_LENGTH * 2 + 1];
    for (int i = 0; i < MD5_DIGEST_LENGTH; i++) {
        sprintf(result + i * 2, "%02x", md5[i]);
    }
    return string(result);
}

// 发送数据
void sendData(char *data, int len, string md5, int sock) {
    // 将数据长度和MD5值打包成头部，并将头部和数据一起发送
    struct {
        uint32_t len;
        char md5[33];
    } header;
    header.len = htonl(len);
    strncpy(header.md5, md5.c_str(), 33);
    send(sock, &header, sizeof(header), 0);
    send(sock, data, len, 0);
}

int main() {
    // 读取文件
    ifstream ifs("test.txt", ios::binary);
    if (!ifs) {
        cout << "File not found." << endl;
        return -1;
    }
    ifs.seekg(0, ifs.end);
    int len = ifs.tellg();
    ifs.seekg(0, ifs.beg);

    // 分配缓冲区
    char *data = new char[len];
    ifs.read(data, len);
    ifs.close();

    // 计算MD5值并发送数据
    string md5 = md5sum(data, len);
    sendData(data, len, md5, 0);

    delete[] data;
    return 0;
}
```



**接收端**

```c++
#include <iostream>
#include <cstring>
#include <openssl/md5.h>
#include <arpa/inet.h>

using namespace std;

// 接收数据
char *receiveData(int sock, int &len, string &md5) {
    // 接收头部，解析数据长度和MD5值
    struct {
        uint32_t len;
        char md5[33];
    } header;
    recv(sock, &header, sizeof(header), 0);
    len = ntohl(header.len);
    md5 = string(header.md5);

    // 分配缓冲区，接收实际数据
    char *data = new char[len];
    recv(sock, data, len, 0);
    return data;
}

// 计算MD5值
string md5sum(char *data, int len) {
    MD5_CTX md5ctx;
    MD5_Init(&md5ctx);
    MD5_Update(&md5ctx, data, len);
    unsigned char md5[MD5_DIGEST_LENGTH];
    MD5_Final(md5, &md5ctx);
    char result[MD5_DIGEST_LENGTH * 2 + 1];
    for (int i = 0; i < MD5_DIGEST_LENGTH; i++) {
        sprintf(result + i * 2, "%02x", md5[i]);
    }
    return string(result);
}

int main() {
    // 接收数据
    int len;
    string md5;
    char *data = receiveData(0, len, md5);

    // 计算MD5值并比较
    string md5recv = md5sum(data, len);
    if (md5recv != md5) {
        cout << "MD5 mismatch." << endl;
    } else {
        cout << "MD5 match." << endl;
    }

    delete[] data;
    return 0;
}
```





### 吞吐量测试

**发送端**

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <chrono>

using namespace std;

// 发送指定大小的数据
void send_data(int sock, int data_size) {
    char *data = new char[data_size];
    memset(data, 'a', data_size);
    int total_sent = 0;
    auto start_time = chrono::steady_clock::now();

    while (total_sent < data_size) {
        int sent = send(sock, data + total_sent, data_size - total_sent, 0);
        if (sent == -1) {
            cerr << "Error: failed to send data." << endl;
            break;
        }
        total_sent += sent;
    }

    auto end_time = chrono::steady_clock::now();
    double time_elapsed = chrono::duration_cast<chrono::microseconds>(end_time - start_time).count() / 1000000.0;
    double throughput = data_size * 8.0 / time_elapsed / 1000000.0;
    cout << "Sent " << data_size << " bytes with throughput " << throughput << " Mbps." << endl;

    delete[] data;
}

int main() {
    int data_size = 1024; // 数据大小，单位为Byte
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if (sock == -1) {
        cerr << "Error: failed to create socket." << endl;
        return -1;
    }

    sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(30000);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (connect(sock, (sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        cerr << "Error: failed to connect to server." << endl;
        return -1;
    }

    while (true) {
        send_data(sock, data_size);
        sleep(1);
    }

    close(sock);

    return 0;
}
```





**接收端**

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <chrono>
#include <iomanip>

using namespace std;

// 接收数据
bool receive_data(int sock, int data_size) {
    char *data = new char[data_size];
    int total_received = 0;

    while (total_received < data_size) {
        int received = recv(sock, data + total_received, data_size - total_received, MSG_WAITALL);
        if (received == -1) {
            cerr << "Error: failed to receive data." << endl;
            delete[] data;
            return false;
        }
        total_received += received;
    }

    delete[] data;
    return true;
}

int main() {
    int data_size = 1024; // 数据大小，单位为Byte
    int loss_count = 0; // 丢包计数器
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if (sock == -1) {
        cerr << "Error: failed to create socket." << endl;
        return -1;
    }

    sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(30000);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sock, (sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        cerr << "Error: failed to bind socket." << endl;
        return -1;
    }

    if (listen(sock, 1) == -1) {
        cerr << "Error: failed to listen on socket." << endl;
        return -1;
    }

    sockaddr_in client_addr;
    socklen_t addr_len = sizeof(client_addr);

    while (true) {
        int client_sock = accept(sock, (sockaddr*)&client_addr, &addr_len);
        if (client_sock == -1) {
            cerr << "Error: failed to accept client connection." << endl;
            continue;
        }

        // 接收数据
        bool res = receive_data(client_sock, data_size);
        close(client_sock);

        if (res) {
            cout << "Received " << data_size << " bytes." << endl;
        } else {
            loss_count++;
            cout << "Packet loss count: " << loss_count << endl;
        }

        auto cur_time = chrono::system_clock::now();
        time_t t = chrono::system_clock::to_time_t(cur_time);
        cout << "Current time: " << put_time(localtime(&t), "%Y-%m-%d %H:%M:%S") << endl;

        // 计算吞吐量
        static int total_received = 0;
        total_received += data_size;
        double time_elapsed = chrono::duration_cast<chrono::microseconds>(cur_time.time_since_epoch()).count() / 1000000.0;
        double throughput = total_received * 8.0 / time_elapsed / 1000000.0;
        cout << "Total received: " << total_received << " bytes, current throughput: " << throughput << " Mbps." << endl;
    }

    close(sock);

    return 0;
}
```





### 时延测试

**发送端**

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <chrono>

using namespace std;

// 发送指定大小的数据并返回接收时间
chrono::steady_clock::time_point send_data(int sock, int data_size) {
    char *data = new char[data_size];
    memset(data, 'a', data_size);

    auto start_time = chrono::steady_clock::now();

    int total_sent = 0;
    while (total_sent < data_size) {
        int sent = send(sock, data + total_sent, data_size - total_sent, 0);
        if (sent == -1) {
            cerr << "Error: failed to send data." << endl;
            break;
        }
        total_sent += sent;
    }

    delete[] data;

    return start_time;
}

int main() {
    int data_size = 1024; // 数据大小，单位为Byte
    int loop_count = 10000; // 循环次数
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if (sock == -1) {
        cerr << "Error: failed to create socket." << endl;
        return -1;
    }

    sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(30000);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (connect(sock, (sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        cerr << "Error: failed to connect to server." << endl;
        return -1;
    }

    double total_delay = 0.0;

    for (int i = 0; i < loop_count; i++) {
        // 发送数据并记录时间
        auto start_time = send_data(sock, data_size);

        // 接收数据并计算时延
        char buf[1];
        int received = recv(sock, buf, 1, MSG_WAITALL);
        if (received == -1) {
            cerr << "Error: failed to receive response." << endl;
            break;
        }

        auto end_time = chrono::steady_clock::now();
        double delay = chrono::duration_cast<chrono::microseconds>(end_time - start_time).count() / 2.0;
        total_delay += delay;

        cout << "Loop " << i + 1 << ", sent " << data_size << " bytes, delay " << delay << " us." << endl;
    }

    double average_delay = total_delay / loop_count;
    cout << "Average delay: " << average_delay << " us." << endl;

    close(sock);

    return 0;
}
```



**接收端**

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

using namespace std;

int main() {
    int data_size = 1024; // 数据大小，单位为Byte
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if (sock == -1) {
        cerr << "Error: failed to create socket." << endl;
        return -1;
    }

    sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(30000);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sock, (sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        cerr << "Error: failed to bind socket." << endl;
        return -1;
    }

    if (listen(sock, 1) == -1) {
        cerr << "Error: failed to listen on socket." << endl;
        return -1;
    }

    sockaddr_in client_addr;
    socklen_t addr_len = sizeof(client_addr);

    while (true) {
        int client_sock = accept(sock, (sockaddr*)&client_addr, &addr_len);
        if (client_sock == -1) {
            cerr << "Error: failed to accept client connection." << endl;
            continue;
        }

        char *data = new char[data_size];
        int total_received = 0;

        while (total_received < data_size) {
            int received = recv(client_sock, data + total_received, data_size - total_received, MSG_WAITALL);
            if (received == -1) {
                cerr << "Error: failed to receive data." << endl;
                delete[] data;
                close(client_sock);
                continue;
            }
            total_received += received;
        }

        // 模拟处理时间，暂停 100us
        usleep(100);

        // 发送响应数据
        char buf[1] = {'a'};
        int sent = send(client_sock, buf, 1, 0);
        if (sent == -1) {
            cerr << "Error: failed to send response." << endl;
        }

        delete[] data;
        close(client_sock);
    }

    close(sock);

    return 0;
}
```

