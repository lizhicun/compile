# 编译优化（上）

### 内容摘要
* 编译和链接过程
* 静态编译和动态编译
* CMake和Make
* 编译耗时分析
* 编译优化手段

### 编译和链接过程
* 单个cpp文件编译过程
```
int g_a = 1;            //定义有初始值全局变量
int g_b;                //定义无初始值全局变量
static int g_c;         //定义全局static变量
extern int g_x;         //声明全局变量
extern int sub();   //函数声明

int sum(int m, int n) {     //函数定义
    return m+n;
}
int main(int argc, char* argv[]) {
    static int s_a = 0;     //局部static变量
    int l_a = 0;                //局部非static变量
    sum(g_a,g_b);
    return 0;
}
```
* 多个cpp文件编译过程
