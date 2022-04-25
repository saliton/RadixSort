
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/RadixSort/blob/main/RadixSort.ipynb)

# まだクイックソート使ってるの？これからは基数ソートの時代

皆さん、[基数ソート](https://ja.wikipedia.org/wiki/%E5%9F%BA%E6%95%B0%E3%82%BD%E3%83%BC%E3%83%88)(Radix Sort)はご存知ですね？O(kN)のすごいやつです。実はMacに搭載されているBSDのsortコマンドは、--radixsortオプションがあったりします。ただし、このコマンドは数値には使えないとマニュアルに書いてあります。もともと文字列用に考案されたものだからでしょうか。そんな中、[こんな記事](https://probablydance.com/2016/12/27/i-wrote-a-faster-sorting-algorithm/)を見つけました。整数はもとより、浮動小数点数でも基数ソート出来るよ〜という記事です。詳しくは記事を読んでいただくとして、この記事にはソースコードが付いています。これは早速ダウンロードしてColab上で性能検証したい！ということでやってみました。

## 乱数の生成

乱数を1億個生成します。


```python
%%time
from random import random

with open('input', 'w') as fout:
    for _ in range(100000000):
        print((random() - 0.5) * 10000, file=fout)
```

    CPU times: user 2min 26s, sys: 3.78 s, total: 2min 30s
    Wall time: 2min 31s


## sortコマンド

比較のため、sortコマンドでソートして、時間を測定します。


```shell
%%time
!sort -n input > output_sort
```

    tcmalloc: large alloc 5459148800 bytes == 0x55e132772000 @  0x7f487d24c1e7 0x55e1310e4718 0x55e1310e35a1 0x7f487cc2ac87 0x55e1310e402a
    CPU times: user 1.12 s, sys: 139 ms, total: 1.26 s
    Wall time: 2min 40s



```shell
!wc -l output_sort
!head output_sort
```

    100000000 output_sort
    -4999.999796334888
    -4999.999574202269
    -4999.999531516728
    -4999.999289444984
    -4999.999184415858
    -4999.9991355508155
    -4999.999037242266
    -4999.998736418958
    -4999.998521971604
    -4999.99851092898


ちゃんとソートできています。

## 浮動小数点数の読み込みライブラリ

入出力で足を引っ張られたくないので、浮動小数点数の読み込みの速度に焦点を当てたライブラリを使いましょう。


```shell
%cd /content/
!wget https://github.com/fastfloat/fast_float/releases/download/v3.4.0/fast_float.h
```

    /content
    --2022-03-31 16:32:46--  https://github.com/fastfloat/fast_float/releases/download/v3.4.0/fast_float.h
    .
    .
    .
    Length: 107579 (105K) [application/octet-stream]
    Saving to: ‘fast_float.h’
    
    fast_float.h        100%[===================>] 105.06K  --.-KB/s    in 0.02s   
    
    2022-03-31 16:32:46 (4.94 MB/s) - ‘fast_float.h’ saved [107579/107579]
    


## std::sort

STLのsortを使ってソートしてみます。


```c
%%writefile /content/std_sort_vector.cpp

#include <stdio.h>
#include <stdlib.h>
#include <chrono>
#include <vector>
#include <algorithm>
#include "fast_float.h"

int main(int argc, char *argv[])
{
    char buf[BUFSIZ];
    std::vector<double> input;
    double f;
    fprintf(stderr, "loading...");
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        fast_float::from_chars(buf, buf + strlen(buf), f);     
        input.push_back(f);
    }
    fprintf(stderr, "done\n");
    auto start = std::chrono::system_clock::now();
    std::sort(input.begin(), input.end());
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "%f sec\n", elapsed / 1000); 
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%lf\n", input[i]);
    }
}
```

    Writing /content/std_sort_vector.cpp



```shell
!g++ -Ofast std_sort_vector.cpp
```


```shell
%%time
!./a.out < input > output_std_vector
```

    loading...tcmalloc: large alloc 1073741824 bytes == 0x564a1d850000 @  0x7f310e038887 0x5649dc1f6694 0x5649dc1f5dd5 0x7f310d692c87 0x5649dc1f5e6a
    done
    14.933000 sec
    CPU times: user 737 ms, sys: 82.9 ms, total: 820 ms
    Wall time: 1min 39s



```shell
!wc -l output_std_vector
!head output_std_vector
```

    100000000 output_std_vector
    -4999.999796
    -4999.999574
    -4999.999532
    -4999.999289
    -4999.999184
    -4999.999136
    -4999.999037
    -4999.998736
    -4999.998522
    -4999.998511


速い！STLのsortは速いと聞いていましたが、浮動小数点数に特化しているのも効いている可能性があります。sortコマンドは高機能な分、余計な処理が挟まっているのかもしれません。

sort部分のみの時間も測定しています。割り当てられるインスタンスにもよりますが概ね12〜15秒です。


## ska sort

それでは基数ソートを測定しましょう。


```shell
%cd /content
!git clone https://github.com/skarupke/ska_sort.git
```

    /content
    Cloning into 'ska_sort'...
    remote: Enumerating objects: 16, done.[K
    remote: Total 16 (delta 0), reused 0 (delta 0), pack-reused 16[K
    Unpacking objects: 100% (16/16), done.



```c
%%writefile /content/ska_sort.cpp
#include <stdio.h>
#include <stdlib.h>
#include <vector>
#include <chrono>
#include "fast_float.h"
#include "ska_sort/ska_sort.hpp"

int main(int argc, char *argv[])
{
    char buf[BUFSIZ];
    std::vector<double> input;
    double f;
    fprintf(stderr, "loading...");
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        fast_float::from_chars(buf, buf + strlen(buf), f);     
        //input.puah_back(atof(buf));
        input.push_back(f);
    }
    fprintf(stderr, "done\n");
    auto start = std::chrono::system_clock::now();
    ska_sort(input.begin(), input.end());
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "%f sec\n", elapsed / 1000);
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%lf\n", input[i]);
    }
}
```

    Overwriting /content/ska_sort.cpp



```shell
!g++ -Ofast ska_sort.cpp
```


```shell
%%time
!./a.out < input > output_ska
```

    loading...tcmalloc: large alloc 1073741824 bytes == 0x55b2e3bb4000 @  0x7f8014c04887 0x55b2a242d464 0x55b2a242cd75 0x7f801425ec87 0x55b2a242ce3a
    done
    5.022000 sec
    CPU times: user 622 ms, sys: 80.7 ms, total: 702 ms
    Wall time: 1min 28s



```shell
!wc -l output_ska
!head output_ska
```

    100000000 output_ska
    -4999.999796
    -4999.999574
    -4999.999532
    -4999.999289
    -4999.999184
    -4999.999136
    -4999.999037
    -4999.998736
    -4999.998522
    -4999.998511


確かに速い。特にsort部分のみの処理時間はSTLのsortの3分の１以下です。ただ、ファイルの読み書きの時間が大部分を占めるので、全体としては10秒ほど、1割ぐらいしか時間は変わりません。それでも:内部で浮動小数点数のソートが必要なプログラムにはとても有効なことが分かりました。積極的に使っていきたいですね。
