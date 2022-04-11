# STL常用组件

### vector, 变长数组，倍增的思想
```cpp
    size()  返回元素个数
    empty()  返回是否为空
    clear()  清空
    front()/back()
    push_back()/pop_back()
    begin()/end()
    [] 即和数组一样，支持随机寻址

初始化：
    vector<vector<int>> matrix(3, vector<int>(5,0));
    vector<int> a(3, 5), b(5,3);
运算符：支持比较运算，按字典序
    if(a > b) cout << " a > b ";
遍历：
    //遍历方法一
    for(auto x:a)   cout << x << ' ';
    //遍历方法二 
    for(int i = 0; i < a.size(); i ++)  cout << a[i] << ' ';
    //遍历方法三 迭代器可以看成是指针
    for(vector <int> :: iterator i = a.begin(); i != a.end(); i ++) cout << *i << ' ';
    for(auto i = a.begin(); i != a.end(); i ++) cout << *i << ' ';
```


### pair
```cpp
pair<int, int>
    first, 第一个元素
    second, 第二个元素
    支持比较运算，以first为第一关键字，以second为第二关键字（字典序）
初始化：
    pair <string , int> p;
    p = {"hello",  20}
    p = make_pair("hello", 20);
访问：
    cout << p.first << ' ' << p.second ;
```

### string
```cpp
string，字符串
    size()/length()  返回字符串长度
    empty()
    clear()
    substr(起始下标，(子串长度))
    erase(position) //删除固定位置
    erase(起始下标，(子串长度)) //erase(1,2) 删除以1为索引，长度为2的字符串
    c_str()  返回字符串所在字符数组的(const类型)起始地址

初始化:
    string a("hello");
    string a = "hello";
取子串：
    a.substr(1,3);//返回下标从1开始且长度为3的子串，包括左端点 
字符串变量和字符数组之间的转化：char ch[] = "hello"; string str = "world";
    ch[] -> str :   str = ch;
    str -> ch[] :   strcpy(ch,str.c_str());
拼接字符串：
    a += "world";//新增字符串
    a.append(" world");//新增字符串
    a.push_back('.');//在字符串末新增单个字符
在字符串指定位置添加字符串
    a.insert(3,"world");
可以使用STL接口，可以理解为一个特殊的容器，容器里装的是的字符
    a.push_back('.');//在字符串末新增单个字符
    a.pop_back();
字符串变量的交换和取代：
    a.swap(str);//str 为字符串变量
    a.replace(1,2,str2) //用字符串str2取代字符串a下标为1长度为2的子串
```


### queue, 队列
```cpp
    size()
    empty()
    push()  向队尾插入一个元素
    front()  返回队头元素
    back()  返回队尾元素
    pop()  弹出队头元素
```


### priority_queue, 优先队列，默认是大根堆
**注意stl class只能用模板类，algrithm可以用模板类和比较函数**
```cpp
    push()  插入一个元素
    top()  返回堆顶元素
    pop()  弹出堆顶元素
定义成小根堆的方式：
    priority_queue<int, vector<int>, greater<int>> q; 或者存为负数
自定义模板类：
    struct Node { int val; };
    struct cmp  {
        bool operator()(Node a, Node b) { return a.val > b.val; }
    }; //定义小于关系,返回逻辑上a<b
    priority_queue<Node, vector<Node>, cmp> q; //这里是泛型，不用()
```


### stack, 栈
```cpp
    size()
    empty()
    push()  向栈顶插入一个元素
    top()  返回栈顶元素
    pop()  弹出栈顶元素
    没有clear 
```



### deque, 双端队列
```cpp
    size()
    empty()
    clear()
    front()/back() 返回第一个或最后一个元素 
    push_back()/pop_back()
    push_front()/pop_front()
    begin()/end()
    []
```


### set, map, multiset, multimap, 基于平衡二叉树（红黑树），动态维护有序序列
区别： set 与 multiset 的区别：set 里面不可以有重复元素，而multiset 可以有
```cpp
    size()
    empty()
    clear()
    begin()/end()
    ++, -- 返回前驱和后继，时间复杂度 O(logn)
```
* **set/multiset**
    ```cpp
        insert()  插入一个数
        find()  查找一个数，返回iter
        count()  返回某一个数的个数，这里只能为1或0
        erase()
            (1) 输入是一个数x，删除所有x   O(k + logn)
            (2) 输入一个迭代器，删除这个迭代器
        lower_bound()/upper_bound()
            lower_bound(x)  返回大于等于x的最小的数的迭代器
            upper_bound(x)  返回大于x的最小的数的迭代器
        erase(st.find(1024));
    ```

* **map/multimap**
    ```cpp
        insert()  插入的数是一个pair
        erase()  输入的参数是pair或者迭代器
        find()
        []  注意下表访问后改值得value会被初始化
            注意multimap不支持此操作。 时间复杂度是 O(logn)
        lower_bound()/upper_bound()
    ```



* **unordered_set, unordered_map, unordered_multiset, unordered_multimap, 哈希表**
    ```cpp
    增删改查的时间复杂度是 O(1)
    不支持 lower_bound()/upper_bound()， 迭代器的++，--
    ```



### bitset, 圧位
```cpp
bitset<10000> s;
~, &, |, ^  支持所有运算 
>>, <<
==, !=
[]  也可以当成数组来用 

count()  返回有多少个1

any()  判断是否至少有一个1
none()  判断是否全为0

set()  把所有位置成1
set(k, v)  将第k位变成v
reset()  把所有位变成0
flip()  等价于~     取反 
flip(k) 把第k位取反
```

### 去重
```cpp
set<int> st(vec.begin(),vec.end());
for(int i:st) cout<<i;
vector<int> b(st.begin(), st.end());


sort(vec.begin(), vec.end());
vec.erase(unique(vec.begin(), vec.end()), vec.end()); //unique返回vector中没有重复元素的下一个位置的迭代器
```

### Algorithm 
```cpp
struct cmp{
    bool operator()(int a,int b){return a > b};
} //返回逻辑上a<b
bool mycmp(int a, int b) { return a > b; }

lower_bound(vec.begin(), vec.end(), n, cmp()); //模板类，这里需要括号
lower_bound(vec.begin(), vec.end(), n, mycmp); 
```

## c++ builtin 位运算函数
```cpp
#include<stdio.h>
int __builtin_popcount(unsigned int n)//判断n的二进制中有多少个1
__builtin_popcount(15) //为4 15=(1111)b

int _builtin_ffs(unsigned int n) //该函数判断n的二进制末尾最后一个1的位置，从1开始
cout<<__builtin_ffs(1)<<endl;//输出1
cout<<__builtin_ffs(8)<<endl;//输出4

__builtin_ctz(unsigned int n) //该函数判断n的二进制末尾后面0的个数，当n为0时，和n的类型有关
_builtin_ctzll(8)//3
_builtin_ctzll(1)//0

__builtin_clz (unsigned int x) //返回前导的0的个数
__builtin_parity(unsigned int n)//判断n的二进制中1的个数的奇偶性

__builtin_clzll(unsigned long long n) //long long，加上ll后缀
```