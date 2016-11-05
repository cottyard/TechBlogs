����� Monad������Ҫ֪����Ϊʲô���ڡ�

����֪�����������Ǹ��ö�������������ͬ������Զ������ͬ�������������
```
a -> b
```
������û�и����ã�Ҳ����˵����û���޸ı������֮������ݡ�

���ԣ������ǵĳ���Ҫ�� IO ������ʱ�����ǲ����ڳ����ﶫһ����һ����д IO ���룬��Ϊ��Щ IO �����޸ĵ����ݣ�������Ļ�Դ棩��û�б����ǵĴ��뽨ģ�ģ�һ�����ˣ��������Ҳ�����ˡ�

����ô���أ����ǿ��԰����뿴�� a����������� b���Ϳ����ô���Ҫ�޸ĵ� IO ���ݱ��һ�������������������������ǵĳ����ڲ��Ϳ��Ա��ִ����ˡ�

����Ϊ�������������Ǻ�ʱ����صģ����Կ��԰���ʱ��˳��������������Ȼ���õ����߸�����Ҫȥ�������ǵĴ����������ϵذ�����ת������������� IO ������������ˡ����� Elm ���ԣ���Ϊ�����������ֲ��� Monad����������ô��ģ���ǰ�� Signal�������ĳ��� Cmd/Sub��������������ģ�ͣ���

�Ǽ�Ȼ�����͹����ˣ���Ҫ Monad ���

��Ϊ�����㡣����ʽ��̵ľ�������ڣ����ǿ����úö�ö�СС������������һ��������������д��������������������� IO ��Ҫ�������ģ�ͣ����Ǿ�Ҫ�ѳ�����������Ҫ IO �ĵط���������������������һ�������ݽṹ���������һ�������ݽṹ��Ȼ������ģ������գ����鷳���ⲻ���衣

����������������
����������дһ������
```
a -> c
```
Ȼ����ͷ������С������
```
a -> b
b -> c
```
����������һ����ˮ����
```
(a -> b) -> (b -> c) -> (a -> c)
```
��������С��������������������Ҫ��
```
a -> c
```
������׹�õġ�

Monad ����������������ֳ����ˣ������Ǹ���ǿ�Ľ�ˮ������IO Monad ����Щ��ˮ֮һ�������������ǲ������԰�С�����������������ڽ���ͬʱ�������ǵ�ͷ����һ���������������ǵ�β������һ����������������յ� IO ���������ÿ�������Լ����� IO ���ݽṹ��

���ԣ�Monad ��ˮ���˰�����С����ͷβ���ƴ�������⣬�����Գû���Щ������飬�𵽷�װһ���ֲ��������á���ͬ�Ľ�ˮ���ò�ͬ������Ҫ�ò�ͬ���ࣨIO��Maybe �ȵȣ�����Щ��ˮ���ֿ�����Ȼ�����ǰѽ�ˮ����� M����ʵ��ʹ�õ��������� M ��������ǵ�Ŀ�꺯�������������
```
a -> M c
```
Ϊʲô a û������ M ���أ���Ϊ a �����ǵ����룬������ʱ����ǹ�����ģ������ǰ�������� c��Ȼ���׽��� M �ﰡ��

�������ǻع�ȥ����ˮ�������ͱ���ˣ�
```
(a -> M b) -> (b -> M c) -> (a -> M c)
```
��ʵ���������������Լ��һ�� a����ɣ�
```
M b -> (b -> M c) -> M c
```
����� Haskell �� Monad �� bind ������

������ǿ�һ�����ӡ����Ƕ���һ����ˮ���� ExcpCtrl���������ˮ�������ĺ��������õ��Ľ��������һ��������ֵ��Ҳ������һ���쳣��������ʵ�����ˮ�� Maybe ���߼���һģһ���ģ����ǰ� Nothing ���ͻ��� Raise ���Ͷ��ѡ�
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
���е�һ������ calc ���� 8 ���� 2���õ� 4���ڶ������� calc_excp ���� 8 ���� 0���õ��쳣��Ȼ�������µļ����û���ˣ������Ȼ���쳣��

�� GHC 8 �����������Ľ��
```
Return 4
Raise
```
Ϊ��˵������ĳ��򾿾�������ʲô����д�˸��ȼ۵� C++ �棺
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
���н����
```
4
exception
```
������ map ����ȡ������ֵ�����пոĳ� lambda �ÿ�һ�㡣

�ܽ᣺Monad ��һ�ֳ��󷽷������ڷ�װһ���ֲ���������Ŀ����Ϊ����װ������