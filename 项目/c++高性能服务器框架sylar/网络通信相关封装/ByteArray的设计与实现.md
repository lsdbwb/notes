# ByteArray的整体设计
ByteArray是基础的字节数组
- 提供基础的序列化和反序列化功能
- 提供大小端转换功能
- 支持数据压缩，写入variant压缩编码的数据

## ByteArray的结构
如下图所示，ByteArray由一个一个固定大小的块构成，块与块之间以单向链表的形式组织在一起
![[Pasted image 20220523151252.png]]

### Node
每一个块对应一个Node,Node的定义如下：
```c++
    struct Node {
        Node(size_t s);
        Node();
        ~Node();

        char* ptr;  // 内存块的起始地址
        Node* next;  // 下一个Node的地址
        size_t size; // 块的大小
    };
```

### ByteArray的成员
成员的实际意义如上图所示
```c++
    size_t m_baseSize;    // 每一块的大小
    size_t m_position;   // 进行读写操作的起始位置
    size_t m_capacity;   // 容量，所有块的size之和
    size_t m_size;     // 已经写入的数据的大小
    int8_t m_endian;   // 大小端模式
    Node* m_root;    // 表示第一个块对应的Node
    Node* m_cur;     // 表示当前正在操作的块对应的Node
```

在对ByteArray进行操作时，必须先设置m_position和m_cur的值指明操作的位置，需要调用以下的setPosition函数
```c++
void ByteArray::setPosition(size_t v) {
	// 只能在m_size前面进行操作，保证是连续的
    if(v > m_size) {
        throw std::out_of_range("set_position out of range");
    }
    // 设置操作的位置 m_posiiton
    m_position = v;
    // 从根节点出发，根据m_position找到所在块，并设置m_cur
    m_cur = m_root;
    while(v > m_cur->size) {
        v -= m_cur->size;
        m_cur = m_cur->next;
    }
    if(v == m_cur->size) {
        m_cur = m_cur->next;
    }
}
```

## ByteArray的使用逻辑
- ByteArray对已经写入的内容没有任何保护机制，你可以任意覆盖写入的内容
- 同样对于读操作也没有限制，可以读0到m_size任一位置的内容
- 唯一限制是要保证ByteArray的连续性，即假如ByteArray已经写入的内容的大小为m_size，如果继续写入时，不允许在超过m_size的位置pos进行写,导致m_size到pos之间没有内容。即块内部不能有碎片

## 读/写流程
### 读ByteArray的操作
```c++
void ByteArray::read(void* buf, size_t size) const {
    if(size > getReadSize()) {
        throw std::out_of_range("not enough len");
    }
	// 根据position找到在cur块的操作的位置npos
    size_t npos = m_position % m_baseSize;
    size_t ncap = m_cur->size - npos;
    size_t bpos = 0;
    Node* cur = m_cur;
    while(size > 0) {
	    // 当前块的从npos位置开始可读的内容足够，则直接将数据拷贝完毕
        if(ncap >= size) {
            memcpy((char*)buf + bpos, cur->ptr + npos, size);
            if(cur->size == (npos + size)) {
                cur = cur->next;
            }
            position += size;
            bpos += size;
            size = 0;
        } else {
        // 当前块的从npos位置开始可读的内容不够，则读完这一块后再从下一块读
            memcpy((char*)buf + bpos, cur->ptr + npos, ncap);
            position += ncap;
            bpos += ncap;
            size -= ncap;
            cur = cur->next;
            ncap = cur->size;
            npos = 0;
        }
    }
}
```
- （重载）`read()`（核心）
功能：将内存块的内容读取到缓冲区中，但不影响当前内存指针指向的位置，使用一个外部传入的内存指针`position`，而不使用当前真正的内存指针`m_position`。即：用户只关心存储的内容，而不关心是否移除内存中的内容，或许还要紧接着写入内容。
`void read(char *buf, size_t size, size_t position) const;`


### 写ByteArray的操作
```c++
void ByteArray::write(const void* buf, size_t size) {
    if(size == 0) {
        return;
    }
    // 首先进行扩容操作，避免ByteArray的容量不够
    addCapacity(size);
	
    size_t npos = m_position % m_baseSize; // 当前块的位置
    size_t ncap = m_cur->size - npos; //当前块剩下的容量
    size_t bpos = 0; 

    while(size > 0) {
        // 当前Node从npos位置开始有足够的空间
        if(ncap >= size) {
	        // 拷贝所有数据
            memcpy(m_cur->ptr + npos, (const char*)buf + bpos, size);
            if(m_cur->size == (npos + size)) {  // 当前块恰好填满
                m_cur = m_cur->next;
            }
            m_position += size;
            bpos += size;
            size = 0;
        // 当前Node从npos位置开始没有足够的空间，先将当前Node占满，再到下一Node处理
        } else {
            memcpy(m_cur->ptr + npos, (const char*)buf + bpos, ncap);
            m_position += ncap; // 更新m_position的位置
            bpos += ncap;
            size -= ncap;
            m_cur = m_cur->next;
            ncap = m_cur->size;
            npos = 0;
        }
    }
	// 更新m_size
    if(m_position > m_size) {
        m_size = m_position;
    }
}
```
### 扩容操作
扩容是以块为单位进行的，因为ByteArray以块为最小单位。
```c++
void ByteArray::addCapacity(size_t size) {
    if(size == 0) {
        return;
    }
    // 容量足够，直接返回
    size_t old_cap = getCapacity();
    if(old_cap >= size) {
        return;
    }
	// 容量不够，进行扩容
    size = size - old_cap;
    // count是扩容的节点的数量
    size_t count = (size / m_baseSize) + (((size % m_baseSize) > old_cap) ? 1 : 0);
    // 找到末尾节点
    Node* tmp = m_root;
    while(tmp->next) {
        tmp = tmp->next;
    }
    
    Node* first = NULL;
    // 新创建Node并且更新容量
    for(size_t i = 0; i < count; ++i) {
        tmp->next = new Node(m_baseSize);
        if(first == NULL) {
            first = tmp->next;
        }
        tmp = tmp->next;
        m_capacity += m_baseSize;
    }

    if(old_cap == 0) {
        m_cur = first;
    }
}
```

# 序列化和反序列化
## 序列化
### 写入定长的基本类型
```c++
    //write定长的基本类型数据的接口
    void writeFint8  (int8_t value);
    void writeFuint8 (uint8_t value);
    void writeFint16 (int16_t value);
    void writeFuint16(uint16_t value);
    void writeFint32 (int32_t value);
    void writeFuint32(uint32_t value);
    void writeFint64 (int64_t value);
    void writeFuint64(uint64_t value);
```
- 实现时都是调用上面的write函数
- 对于int8，因为只有一个字节8位，大小端的表示都是一样的，直接调用write函数即可
```c++
void ByteArray::writeFint8  (int8_t value) {
    write(&value, sizeof(value));
}
```
- 对于非int8的其他类型，都超过了1个字节的大小，因此要注意大小端的转换
```c++
void ByteArray::writeFint32 (int32_t value) {
    if(m_endian != SYLAR_BYTE_ORDER) {
        value = byteswap(value);
    }
    write(&value, sizeof(value));
}
```

### 写入variant的压缩编码数据
- 接口
```c++
	void writeInt32  (int32_t value);
    void writeUint32 (uint32_t value);
    void writeInt64  (int64_t value);
    void writeUint64 (uint64_t value);

    void writeFloat  (float value);
    void writeDouble (double value);
```
- 写入经过varint压缩编码的uint32_t型数据/int32_t型数据，处理的时候避免负数压缩
```c++
void ByteArray::writeInt32  (int32_t value) {
    writeUint32(EncodeZigzag32(value));
}
// int32_t，先进行一下处理，转变成uint32_t
static uint32_t EncodeZigzag32(const int32_t& v) {
    if(v < 0) {
        return ((uint32_t)(-v)) * 2 - 1;
    } else {
        return v * 2;
    }
}
```
- 使用zigzag压缩算法[[zigzag算法]]对uint32_t类型进行压缩
```c++
void ByteArray::writeUint32 (uint32_t value) {
    uint8_t tmp[5];
    uint8_t i = 0;
    // 每一字节的最高位为标志位
    while(value >= 0x80) {
        tmp[i++] = (value & 0x7F) | 0x80;
        value >>= 7;
    }
    tmp[i++] = value;
    write(tmp, i);
}
```

## 反序列化
### 读定长的基本类型
除了1字节长度的数据，其他 >1字节的数据需要检查**存储的数据字节序是否和用户物理机的字节序相同**，不同需要进行字节序调整交换
- 接口
```c++
/*固定长度读取*/
int8_t      readFint8();
uint8_t     readFuint8();
int16_t     readFint16();
uint16_t    readFuint16();
int32_t     readFint32();
uint32_t    readFuint32();
int64_t     readFint64();
uint64_t    readFuint64();

float       readFloat();
double      readDouble();
```
- 接口实现
```c++
int8_t ByteArray::readFint8()
{
    int8_t v;
    read((char*)&v, sizeof(v));
    return v;
}


uint8_t ByteArray::readFuint8()
{
    uint8_t v;
    read((char*)&v, sizeof(v));
    return v;
}

#define READ_XX(type)\
    type v;\
    read((char*)&v, sizeof(v));\
    if(m_endian !=  KIT_BYTE_ORDER)\
        v = byteswap(v);\
    return v;


int16_t ByteArray::readFint16()
{
    READ_XX(int16_t);
}
```

### 读variant的压缩编码数据
- 接口
```c++
/*读取varint压缩后的数据*/
int32_t     readInt32();
uint32_t    readUint32();
int64_t     readInt64();
uint64_t    readUint64();
```
- 接口实现
```c++
// 先按uint32_t读，读完后恢复为int32_t
int32_t  ByteArray::readInt32() {
    return DecodeZigzag32(readUint32());
}

uint32_t ByteArray::readUint32() {
    uint32_t result = 0;
    for(int i = 0; i < 32; i += 7) {
	    // 读一字节数据
        uint8_t b = readFuint8();
        // 根据每一字节的最高位
        if(b < 0x80) {
            result |= ((uint32_t)b) << i;
            break;
        } else {
            result |= (((uint32_t)(b & 0x7f)) << i);
        }
    }
    return result;
}
```

# 大小端转换
大小端转换的模板函数
```c++
template<class T>
typename std::enable_if<sizeof(T) == sizeof(uint32_t), T>::type
byteswap(T value) {
    return (T)bswap_32((uint32_t)value);
}
```