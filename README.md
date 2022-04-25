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

    CPU times: user 2min 22s, sys: 4.74 s, total: 2min 26s
    Wall time: 2min 27s


## sortコマンド

比較のため、sortコマンドでソートして、時間を測定します。


```shell
%%time
!sort -n input > output_sort
```

    tcmalloc: large alloc 7254646784 bytes == 0x55ae84d60000 @  0x7fd703e181e7 0x55ae8347a718 0x55ae834795a1 0x7fd7037f6c87 0x55ae8347a02a
    CPU times: user 1.25 s, sys: 164 ms, total: 1.41 s
    Wall time: 2min 52s



```shell
!wc -l output_sort
!head output_sort
```

    100000000 output_sort
    -4999.999748075656
    -4999.999715955036
    -4999.999667609005
    -4999.99955786198
    -4999.999426697782
    -4999.999419544308
    -4999.999309093983
    -4999.999102087969
    -4999.999082534726
    -4999.998984853804


ちゃんとソートできています。

## 浮動小数点数の読み込みライブラリ

入出力で足を引っ張られたくないので、浮動小数点数の変換の速度に焦点を当てたライブラリを使いましょう。


```shell
%cd /content/
!wget https://github.com/fastfloat/fast_float/releases/download/v3.4.0/fast_float.h
```

    /content
    --2022-04-23 19:04:21--  https://github.com/fastfloat/fast_float/releases/download/v3.4.0/fast_float.h
    Resolving github.com (github.com)... 140.82.113.3
    Connecting to github.com (github.com)|140.82.113.3|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/305438763/cf03fab1-3b3d-4624-915d-43783c554d5b?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20220423%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220423T190422Z&X-Amz-Expires=300&X-Amz-Signature=3f6845fbf3c9409057ef1f9b9244a877cf7d091ff72e531c61b2243cfab539ec&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=305438763&response-content-disposition=attachment%3B%20filename%3Dfast_float.h&response-content-type=application%2Foctet-stream [following]
    --2022-04-23 19:04:22--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/305438763/cf03fab1-3b3d-4624-915d-43783c554d5b?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20220423%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220423T190422Z&X-Amz-Expires=300&X-Amz-Signature=3f6845fbf3c9409057ef1f9b9244a877cf7d091ff72e531c61b2243cfab539ec&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=305438763&response-content-disposition=attachment%3B%20filename%3Dfast_float.h&response-content-type=application%2Foctet-stream
    Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
    Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.108.133|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 107579 (105K) [application/octet-stream]
    Saving to: ‘fast_float.h’
    
    fast_float.h        100%[===================>] 105.06K  --.-KB/s    in 0.02s   
    
    2022-04-23 19:04:22 (5.14 MB/s) - ‘fast_float.h’ saved [107579/107579]
    


## std::sort

STLのsortを使ってソートしてみます。


```cpp
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
 
    auto start = std::chrono::system_clock::now();
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        fast_float::from_chars(buf, buf + strlen(buf), f);     
        input.push_back(f);
    }
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "load: %f sec\n", elapsed / 1000);
 
    start = std::chrono::system_clock::now();
    std::sort(input.begin(), input.end());
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "sort: %f sec\n", elapsed / 1000); 
 
    start = std::chrono::system_clock::now();
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%.12lf\n", input[i]);
    }
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "out:  %f sec\n", elapsed / 1000); 
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

    tcmalloc: large alloc 1073741824 bytes == 0x55a9d5cd8000 @  0x7fd6e11ea887 0x55a9941fd6e4 0x55a9941fce25 0x7fd6e0844c87 0x55a9941fceba
    load: 10.923000 sec
    sort: 13.579000 sec
    out:  105.370000 sec
    CPU times: user 945 ms, sys: 108 ms, total: 1.05 s
    Wall time: 2min 10s



```shell
!wc -l output_std_vector
!head output_std_vector
```

    100000000 output_std_vector
    -4999.999748075656
    -4999.999715955036
    -4999.999667609005
    -4999.999557861980
    -4999.999426697782
    -4999.999419544308
    -4999.999309093983
    -4999.999102087969
    -4999.999082534726
    -4999.998984853804


まあまあ速い。STLのsortは速いと聞いていましたが、浮動小数点数に特化しているのも効いている可能性があります。sortコマンドは高機能な分、余計な処理が挟まっているのかもしれません。

sort部分のみの時間も測定しています。割り当てられるインスタンスにもよりますが概ね12〜15秒です。

やっぱりロードの時間も気になるので、浮動小数点数変換の他の方法でも測ってみます。

### std::stod()


```cpp
%%writefile /content/std_sort_vector.cpp

#include <stdio.h>
#include <stdlib.h>
#include <chrono>
#include <vector>
#include <algorithm>
#include <string>

int main(int argc, char *argv[])
{
    char buf[BUFSIZ];
    std::vector<double> input;
 
    auto start = std::chrono::system_clock::now();
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        input.push_back(std::stod(buf));
    }
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "load: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    std::sort(input.begin(), input.end());
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "sort: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%.12lf\n", input[i]);
    }
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "out:  %f sec\n", elapsed / 1000); 
}
```

    Overwriting /content/std_sort_vector.cpp



```shell
!g++ -Ofast std_sort_vector.cpp
```


```shell
%%time
!./a.out < input > output_std_vector
```

    tcmalloc: large alloc 1073741824 bytes == 0x556b8ef2a000 @  0x7f2128388887 0x556b4cf40414 0x556b4cf4000d 0x7f21279e2c87 0x556b4cf400da
    load: 28.063000 sec
    sort: 13.611000 sec
    out:  109.469000 sec
    CPU times: user 1.13 s, sys: 149 ms, total: 1.28 s
    Wall time: 2min 31s


### strtod()


```cpp
%%writefile /content/std_sort_vector.cpp

#include <stdio.h>
#include <stdlib.h>
#include <chrono>
#include <vector>
#include <algorithm>

int main(int argc, char *argv[])
{
    char buf[BUFSIZ];
    std::vector<double> input;
 
    auto start = std::chrono::system_clock::now();
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        input.push_back(strtod(buf, NULL));
    }
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "load: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    std::sort(input.begin(), input.end());
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "sort: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%.12lf\n", input[i]);
    }
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "out:  %f sec\n", elapsed / 1000); 
}
```

    Overwriting /content/std_sort_vector.cpp



```shell
!g++ -Ofast std_sort_vector.cpp
```


```shell
%%time
!./a.out < input > output_std_vector
```

    tcmalloc: large alloc 1073741824 bytes == 0x5577d7c54000 @  0x7f11c2f1b887 0x5577961b00c4 0x5577961afcfb 0x7f11c2575c87 0x5577961afd8a
    load: 26.642000 sec
    sort: 14.598000 sec
    out:  106.754000 sec
    CPU times: user 1.13 s, sys: 126 ms, total: 1.26 s
    Wall time: 2min 28s


### atof()


```cpp
%%writefile /content/std_sort_vector.cpp

#include <stdio.h>
#include <stdlib.h>
#include <chrono>
#include <vector>
#include <algorithm>

int main(int argc, char *argv[])
{
    char buf[BUFSIZ];
    std::vector<double> input;
 
    auto start = std::chrono::system_clock::now();
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        input.push_back(atof(buf));
    }
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "load: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    std::sort(input.begin(), input.end());
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "sort: %f sec\n", elapsed / 1000); 

    start = std::chrono::system_clock::now();
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%.12lf\n", input[i]);
    }
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "out:  %f sec\n", elapsed / 1000); 
}
```

    Overwriting /content/std_sort_vector.cpp



```shell
!g++ -Ofast std_sort_vector.cpp
```


```shell
%%time
!./a.out < input > output_std_vector
```

    tcmalloc: large alloc 1073741824 bytes == 0x55931f5ee000 @  0x7f3960055887 0x5592dd52d0c4 0x5592dd52ccfb 0x7f395f6afc87 0x5592dd52cd8a
    load: 25.384000 sec
    sort: 14.642000 sec
    out:  106.167000 sec
    CPU times: user 1.05 s, sys: 131 ms, total: 1.18 s
    Wall time: 2min 26s


どれもデータのロードに２倍以上の時間がかかっていますね。

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



```cpp
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
 
    auto start = std::chrono::system_clock::now();
    while (fgets(buf, BUFSIZ, stdin) != NULL) {
        fast_float::from_chars(buf, buf + strlen(buf), f);
        input.push_back(f);
    }
    auto end = std::chrono::system_clock::now();
    double elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "load: %f sec\n", elapsed / 1000);

    start = std::chrono::system_clock::now();
    ska_sort(input.begin(), input.end());
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "sort: %f sec\n", elapsed / 1000);
 
    start = std::chrono::system_clock::now();
    for (size_t i = 0; i < input.size(); i++)
    {
        printf("%.12lf\n", input[i]);
    }
    end = std::chrono::system_clock::now();
    elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count();
    fprintf(stderr, "out:  %f sec\n", elapsed / 1000); 
}
```

    Writing /content/ska_sort.cpp



```shell
!g++ -Ofast ska_sort.cpp
```


```shell
%%time
!./a.out < input > output_ska
```

    tcmalloc: large alloc 1073741824 bytes == 0x56204dde2000 @  0x7fc201fe1887 0x56200d2e24b4 0x56200d2e1dbd 0x7fc20163bc87 0x56200d2e1e8a
    load: 10.048000 sec
    sort: 5.174000 sec
    out:  107.086000 sec
    CPU times: user 862 ms, sys: 133 ms, total: 995 ms
    Wall time: 2min 2s



```shell
!wc -l output_ska
!head output_ska
```

    100000000 output_ska
    -4999.999748075656
    -4999.999715955036
    -4999.999667609005
    -4999.999557861980
    -4999.999426697782
    -4999.999419544308
    -4999.999309093983
    -4999.999102087969
    -4999.999082534726
    -4999.998984853804


確かに速い。sort部分の処理時間はSTLのsortの半分以下です。ただ、ファイルの読み書きの時間が大部分を占めるので、全体としては10秒ほど、1割ぐらいしか時間は変わりません。それでも内部で浮動小数点数のソートが必要なプログラムにはとても有効なことが分かりました。大規模データのソートなどには積極的に使っていきたいですね。
