想理解 Monad，首先要知道它为什么存在。

我们知道，纯函数是个好东西，它对于相同输入永远产生相同的输出，记作：
```
a -> b
```
纯函数没有副作用，也就是说，它没有修改本身代码之外的数据。

所以，当我们的程序要做 IO 操作的时候，我们不能在程序里东一个西一个地写 IO 代码，因为这些 IO 代码修改的数据（比如屏幕显存）是没有被我们的代码建模的，一旦改了，代码就再也不纯了。

那怎么办呢？我们可以把输入看作 a，把输出看作 b，就可以让代码要修改的 IO 数据变成一个函数的输入和输出，这样我们的程序内部就可以保持纯洁了。

又因为输入和输出数据是和时间相关的，所以可以按照时间顺序看作是两个流，然后让调用者根据需要去调用我们的纯函数，不断地把输入转换成输出，这样 IO 程序就跑起来了。比如 Elm 语言，作为纯函数语言又不用 Monad，它就是这么搞的（以前用 Signal，后来改成了 Cmd/Sub，但都是这样的模型）。

那既然这样就够用了，还要 Monad 干嘛？

因为不方便。函数式编程的精髓就在于，我们可以用好多好多小小函数，搭搭搭，组成一个个大函数，最终写出整个程序来。如果所有 IO 都要用上面的模型，我们就要把程序里所有需要 IO 的地方都汇总起来，输入做成一个大数据结构，输出做成一个大数据结构，然后再往模型上面凑，很麻烦，这不精髓。

理想的情况是这样：
比如我们想写一个函数
```
a -> c
```
然后手头有两个小函数：
```
a -> b
b -> c
```
于是我们用一个胶水函数
```
(a -> b) -> (b -> c) -> (a -> c)
```
把那两个小函数胶起来，做成我们要的
```
a -> c
```
这样是坠好的。

Monad 的作用在这里就体现出来了，它就是个加强的胶水函数。IO Monad 是这些胶水之一，有了它，我们不但可以把小函数胶起来，还能在胶的同时，把他们的头连成一个输入流，把它们的尾巴连成一个输出流，构成最终的 IO 函数，不用吭哧吭哧自己构造 IO 数据结构。

所以，Monad 胶水除了把两个小函数头尾相接拼起来以外，还可以趁机做些别的事情，起到封装一部分操作的作用。不同的胶水作用不同，所以要用不同的类（IO、Maybe 等等）把这些胶水区分开来。然后，我们把胶水类记作 M，把实际使用的数据塞在 M 里。于是我们的目标函数变成了这样：
```
a -> M c
```
为什么 a 没有套在 M 里呢？因为 a 是我们的输入，它来的时候就是光溜溜的，是我们把它变成了 c，然后套进了 M 里啊。

现在我们回过去看胶水函数，就变成了：
```
(a -> M b) -> (b -> M c) -> (a -> M c)
```
其实它的输入输出可以约掉一个 a，变成：
```
M b -> (b -> M c) -> M c
```
这就是 Haskell 里 Monad 的 bind 函数。

最后我们看一个例子。我们定义一个胶水叫做 ExcpCtrl，用这个胶水胶起来的函数，最后得到的结果可能是一个正常的值，也可能是一个异常。所以其实这个胶水和 Maybe 的逻辑是一模一样的，就是把 Nothing 类型换成 Raise 类型而已。
```Haskell
data ExcpCtrl a = Raise | Return a deriving Show

raise = Raise

instance Monad ExcpCtrl where
    return = Return
    p >>= q = case p of
                Raise -> Raise
                Return x -> q x

calc :: ExcpCtrl Int
calc = do
    a <- return 8
    b <- return 2
    if b == 0 then raise else return $ quot a b

calc_excp :: ExcpCtrl Int
calc_excp = do
    a <- return 8
    b <- return 0
    if b == 0 then raise else return $ quot a b
    return $ quot 8 2

main :: IO ()
main = do
    putStrLn $ show calc
    putStrLn $ show calc_excp
```
其中第一个函数 calc 计算 8 除以 2，得到 4，第二个函数 calc_excp 计算 8 除以 0，得到异常，然后再往下的计算就没用了，结果仍然是异常。

用 GHC 8 运行这个程序的结果
```
Return 4
Raise
```
为了说明上面的程序究竟发生了什么，我写了个等价的 C++ 版：
```C++
#include <iostream>
#include <map>
using namespace std;

template <typename T>
class M
{
public:
    enum Type {
        excp,
        ret
    };
    template <typename T> M(T val, Type t)
    {
        value = val;
        type = t;
    }
    void print()
    {
        if (type == M::ret)
        {
            cout << value;
        }
        else
        {
            cout << "exception";
        }
        cout << endl;
    }
    template <typename T, typename U>
    friend M<U> bind(M<T> m1, M<U>(*morph)(T));
private:
    Type type;
    T value;
};

template <typename T>
M<T> unit(T val)
{
    return M<T>(val, M<T>::ret);
}

template <typename T>
M<T> raise()
{
    return M<T>(T(), M<T>::excp);
}

template <typename T, typename U>
M<U> bind(M<T> m1, M<U>(*morph)(T))
{
    if (m1.type == M<T>::excp)
    {
        return raise<U>();
    }
    else
    {
        return morph(m1.value);
    }
}

map<char, int> closure;

M<int> case1_step2(int res)
{
    closure.insert(pair<char, int>('b', res));
    if (closure.find('b')->second == 0)
    {
        return raise<int>();
    }
    return unit(
        closure.find('a')->second /
        closure.find('b')->second
    );
}

M<int> case1_step1(int res)
{
    closure.insert(pair<char, int>('a', res));
    return bind(
        unit(2),
        case1_step2
    );
}

M<int> case2_step3(int res)
{
    return unit(4);
}

M<int> case2_step2(int res)
{
    closure.insert(pair<char, int>('b', res));
    if (closure.find('b')->second == 0)
    {
        return raise<int>();
    }
    return bind(
        unit(
            closure.find('a')->second /
            closure.find('b')->second),
        case2_step3
    );
}

M<int> case2_step1(int res)
{
    closure.insert(pair<char, int>('a', res));
    return bind(
        unit(0),
        case1_step2
    );
}

int main()
{
    bind(
        unit(8),
        case1_step1
    ).print(); // 4

    closure.clear();

    bind(
        unit(8),
        case2_step1
    ).print(); // exception

    return 0;
}
```
运行结果：
```
4
exception
```
这里用 map 来存取变量的值，等有空改成 lambda 好看一点。

总结：Monad 是一种抽象方法，用于封装一部分操作，最终目的是为了组装函数。