---
title: "天鳳の牌譜から牌山を再現する(Python版)"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "天鳳", "牌譜", "牌山"]
published: true
published_at: "2024-12-02 20:00"
---

## 概要
http://integral001.blog.fc2.com/blog-entry-42.html
integral001氏によるCのコードをPythonに移植した。
ついでにざっくりと解説を加えた。

## 牌山を再現するコード
牌譜のシードから牌山を生成するコードが以下。
ちなみに、シードとは牌譜の`<SHUFFLE seed="mt19937ar-sha512-n288-base64,XXXXXX>`というタグの`XXXXXX`の部分である。

```python:yama_generator.py
import base64
import hashlib
import struct

from mt19937ar import MT19937ar

class YamaGenerator:
    mt: MT19937ar

    __slots__ = ("mt",)

    def __init__(self, b64seed: str) -> None:
        N_INIT = 624

        seed = base64.b64decode(b64seed)
        init = list(struct.unpack(f"<{N_INIT}I", seed))

        mt = MT19937ar()
        mt.init_by_array(init)
        self.mt = mt

    def generate(self) -> list[int]:
        N_SRC = 288
        src: list[int] = []
        for i in range(N_SRC):
            src.append(self.mt.genrand_int32())

        N_RND = 136  # Actually 144
        rnd: list[int] = []
        for i in range(N_SRC // 32):
            s = struct.pack("<32I", *src[32 * i : 32 * (i + 1)])
            m = hashlib.sha512(s).digest()
            rnd.extend(struct.unpack("<16I", m))

        yama = list(range(N_RND))
        for i in range(N_RND):
            j = rnd[i] % (N_RND - i) + i
            yama[i], yama[j] = yama[j], yama[i]

        return yama
```

上記の`YamaGenerator`クラスを使って牌山を再現するコードが以下である。
なお、シードは[つのだ氏のブログ記事](http://blog.tenhou.net/article/174202532.html)に記載されているものを使用した。
```python:main.py
seed = "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"
yama_generator = YamaGenerator(seed)
print(yama_generator.generate())

## [22, 91, 36, 115, 56, 19, 60, 16, 124, ...]
```

出力結果がつのだ氏のブログ記事に記載されている結果と一致することを確認した。


## ざっくり解説
大まかな流れは以下の通り。
1. 牌譜のシードから乱数生成器の初期化用リストを生成
2. 生成したリストを用いて乱数生成器を初期化
3. 288個の乱数を生成
4. 生成した乱数からシャッフル用の事前データを生成
5. Fisher-Yates法でシャッフル
6. 次の局があれば3~5を繰り返し

## 1. 牌譜のシードから乱数生成器の初期化用リストを生成
牌譜のシードはBase64方式でエンコードされているので`base64`モジュールを使ってデコードする。シードをデコードすると2596byteのバイト列になる。

このバイト列を4byteずつ624個のリストに分割する。このとき同時にエンディアンも変換する。`struct`モジュールを利用して、`init = struct.unpack(f"<624I", seed)`のように書くことで、リストの624分割とエンディアンの変換を同時におこなうことができる。

## 2. 生成したリストを用いて乱数生成器を初期化
乱数生成器には`mt19937ar`を使用する。`mt19937ar`は乱数生成のアルゴリズムとしてメルセンヌ・ツイスタを使用している。Pythonの`random`モジュールもメルセンヌ・ツイスタを使用しているらしいが、実装が`mt19937ar`とは異なるようで(深く検証はしていないので適当発言の可能性あり)、少なくとも自分は`random`モジュールを使って牌山を再現することはできなかった。

Cによる`mt19937ar`のコードは[Mersenne Twister with improved initialization (2002)](http://www.math.sci.hiroshima-u.ac.jp/m-mat/MT/MT2002/mt19937ar.html)に掲載されている。Pythonによるコードは発見できなかったので、Cのコードを移植したものを[PyPI](https://pypi.org/project/mt19937ar/)で公開した。
```bash
pip install mt19973ar
```
とすることでインストールできる。GitHubのリンクは[こちら](https://github.com/zurukumo/mt19937ar)。

`mt19937ar`のインスタンスを生成し、1.で生成した`init`を用いて`mt.init_by_array(init)`とすることで、シードさえ同じであれば常に同じ乱数列を生成することができる。

## 3. 288個の乱数を生成
1局分の山の生成には288個の乱数を必要とする。1局目には乱数1 ~ 乱数288を使用、2局目には乱数289 ~ 乱数576を使用、...というように1つの局に対して288個ずつ乱数を使っていく。

なぜ288個なのかは4.で説明する。

`mt.genrand_int32()`を288回を呼び出してリスト`src`に乱数を追加していく。`genrand_int32`で生成される乱数は32bit = 4byteの整数なので、`src`の全体のサイズは4byte * 288個 = 1152byteとなる。

## 4. 生成した乱数からシャッフル用の事前データを生成
乱数のリストである`src`から要素を32個ずつ取り出してバイト列に変換する。このとき同時にエンディアンも変換する。`struct`モジュールを利用して、`s = struct.pack("<32I", *src[32 * i : 32 * (i + 1)])`のように書くことで、バイト列への変換とエンディアンの変換を同時におこなうことができる。

変数`s`は4byteの整数を32個ずつまとめたバイト列なので、全体のサイズは4byte * 32個 = 128byteになる。

次にSHA-512を利用して変数`s`からハッシュ値を生成する。`hashlib`モジュールを利用して、`m = hashlib.sha512(s).digest()`のように書くことでハッシュ値は生成できる。

SHA-512はどんな長さのバイト列を入力として与えても、512bit = 64byteのハッシュ値を生成するので、`m`のサイズは64byteになる。

この`m`を4byteずつ16個のリストに分割する。このとき同時にエンディアンも変換する。`struct`モジュールを利用して、`struct.unpack("<16I", m)`のように書くことで、リストの16分割とエンディアンの変換を同時におこなうことができる。分割したリストをリスト`rnd`に繋げていく。

`s`から`m`を生成する過程で126byteのバイト列を64byteのバイト列にしているので、`src`が4byte * 288個のデータであることから、`rnd`は4byte * 144個のデータになることが計算できる。

麻雀の牌は136枚あるので、シャッフルするために必要な`rnd`の長さは144で十分である。ここから逆算して3.では288個の乱数を生成している。

SHA-512を使う以上、64byte = 4byte * 16個 の単位でデータが生成されることは決まっているので、16の倍数で136を超える最小の数である144がrndの長さになるというわけである。

## 5. Fisher-Yates法でシャッフル
`rnd`を使ってFisher-Yates法でシャッフルする。Fisher-Yates法は以下のように実装できる。
詳細は[フィッシャー–イェーツのシャッフル(Wikipedia)](https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A3%E3%83%83%E3%82%B7%E3%83%A3%E3%83%BC%E2%80%93%E3%82%A4%E3%82%A7%E3%83%BC%E3%83%84%E3%81%AE%E3%82%B7%E3%83%A3%E3%83%83%E3%83%95%E3%83%AB)。

## 6. 次の局があれば3~5を繰り返し
終局するまで3~5を繰り返す。

## 一言
牌山を生成する過程がなぜここまで複雑なのかはよく調べていないのだが、天鳳は牌操作がないことを検証できる仕組みを持っているので、おそらくそれに関係して牌山生成の過程も複雑になっているのだと予想している。

エンディアンの変換などは普段Pythonを使っているとあまり意識しないので勉強になった。

## 参考
http://integral001.blog.fc2.com/blog-entry-42.html
http://blog.tenhou.net/article/174202532.html
http://www.math.sci.hiroshima-u.ac.jp/m-mat/MT/MT2002/mt19937ar.html
https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A3%E3%83%83%E3%82%B7%E3%83%A3%E3%83%BC%E2%80%93%E3%82%A4%E3%82%A7%E3%83%BC%E3%83%84%E3%81%AE%E3%82%B7%E3%83%A3%E3%83%83%E3%83%95%E3%83%AB
