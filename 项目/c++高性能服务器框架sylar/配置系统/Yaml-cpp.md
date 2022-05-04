# YAML简介
YAML是YAML Ain’t Markup Language”的递归表述，Yaml是专门用来作为配置文件的语言。
**语法特点**
1. 大小写敏感
2. 使用缩进表示层级关系
3. 缩进时不允许使用Tab键，只允许使用空格。
4. 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
**支持数据类型**
1. **纯量**（scalars）：单个的、不可再分的值
2. **数组**：一组按次序排列的值，又称为序列（sequence） / 列表（list）
3. **对象**：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
**YAML 对象**
对象键值对使用冒号结构表示 **key: value**，冒号后面要加一个空格。
也可以使用 **key:{key1: value1, key2: value2, ...}**。
还可以使用缩进表示层级关系；
key: 
    child-key: value
    child-key2: value2
**一个相对复杂的例子**
companies:
    -
        id: 1
        name: company1
        price: 200W
    -
        id: 2
        name: company2
        price: 500W
意思是 companies 属性是一个数组，每一个数组元素又是由 id、name、price 三个属性构成。

# Yaml-cpp简介
Yaml-cpp是c++编写的YAML语言的解析和编码库。能够反序列化并解析YAML文件为C++对象，也能将C++对象序列化到YAML文件中。

# 使用方法
