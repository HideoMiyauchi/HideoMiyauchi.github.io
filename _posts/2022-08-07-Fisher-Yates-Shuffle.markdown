---
layout: post
title: Fisher-Yates shuffle
date: 2022-08-07 10:00:00 +0900
description: Fisher Yates Shuffle Algorithm on android
img: 20220807/FisherYatesShuffle.jpg
categories: [Android]
tags: [Android]
---
<!--*****************************************************-->
## Overview
Android上で画像をモザイク化して、それを復元(逆モザイク化)する簡単なアプリを作りました。
<br>
モザイク化は画像を細かなパーツに分割し並び替えています。
並び替えのアルゴリズムは Fisher-Yates shuffle[^footnote]のアルゴリズムを採用しています。
そして、Fisher-Yates shuffleをJNI(Java Native Interface)を通したネイティブコード(C/C++)
で実装し、より高速化を図っています。
<br>
復元はモザイク処理時に使用した並び替え順情報を使って復元しています。
従って、復元にはモザイク化した際に使用した並び替え順情報が必要です。
<br>
これは人工知能を使ってモザイクを消すような先端技術ではありません。

## 画像の暗号化の問題点
元々、画像を他人に見られないように暗号化したかったのですが、
既存の暗号技術で暗号化すると、それはもう画像ではなく単なる数値の羅列に
なってしまいます。同時に拡張子もjpeg, pngとは別のものになります。
また、暗号化した画像をGoogle Driveにアップロードすると"ファイル"として扱われてしまい、
Google Driveの画像プレビュー機能さえ使えず、単なる無機質なファイルと表示されてしまいます。

## 画像のモザイク化のメリット
何の画像だろう？とハッキリ判らなくても、何となく色目や全体の雰囲気から察することができるほうが、
間違った暗号化ファイルを誰かに送りつけるような失敗が減ったり、送られた側もいちいち復号化して内容を確認する手間が省けます。
さらに、モザイク後でも画像フォーマット変換、非過逆圧縮することができます。
画像は画像のままで扱えるほうが各情報システムの内部処理においても(MimeTypeなど)処理を組むことが
できるので便利です。
<br>
では、モザイク化によって、どれだけ認識しにくくなるのかを調べてみます。
<br>
左が元の画像、真ん中が(1セル=50ピクセル)、右が(1セル=10ピクセル)です。

![photo1](/assets/img/20220807/baseImage.png){: width="230" .right}
![photo2](/assets/img/20220807/shuffled50.png){: width="230" .right}
![photo3](/assets/img/20220807/shuffled10.png){: width="230" .right}

50ピクセルぐらいだと元の画像を推測できますが、Google画像検索では引っかからないと思います。
ジグソーパズルが得意な人なら手作業で復元できそうです。
10ピクセルにすれば、ほとんど何の画像か判らなくなりますし、手作業での復元はまず無理でしょう。
このあたりは用途に応じて調整すれば良いと思います。
<br>
言うまでもありませんが、完全なる秘匿を要する場合には既存の暗号化技術を用いるべきです。

<!--*****************************************************-->
## Source Code
Source code may be found as the Kotlin and C/C++ program in the GitHub repository.

[github.com/HideoMiyauchi/FisherYatesShuffle](https://github.com/HideoMiyauchi/FisherYatesShuffle)

<!--*****************************************************-->
## Usage

### 準備
githubよりダウンロードして、Android Studioでプロジェクトを開いてください。

### 実行

①  起動時の画面です。元の画像を最初に表示します。
<br>
②  左上のSTARTボタンを押すとモザイク処理を開始し、モザイク化された画像を２番目に表示します。
<br>
③  続けて再びSTARTボタンを押すと、②の画像に対して復元処理を開始し、結果を画像を３番目に表示します。
<br>

①
![photo4](/assets/img/20220807/step1.png){: width="180" .right}
②
![photo5](/assets/img/20220807/step2.png){: width="180" .right}
③
![photo6](/assets/img/20220807/step3.png){: width="180" .right}

<!--*****************************************************-->
## プログラムの説明

Fisher-Yates shuffleの実装は、次のような呼び出しになっています。
```
① fisherYatesShuffle() [Kotlin]
        ⇓
② jniFisherYatesShuffle() [JNI]
        ⇓
③ FisherYatesShuffle() [C/C++]
```
まず、fisherYatesShuffle関数から説明します。
<br>
fisherYatesShuffle関数の1番目の引数は処理を施したいビットマップを指定します。
2番目の引数はfalseでモザイク処理、trueで復元処理を指定します。
変数randomSequenceが並び替え順情報で、とても大切です。
この情報を失うと復元できなくなるので注意してください。
ここではIntArrayコレクションのshuffle関数を引数無しで使用していますが、
ランダム性を上げる為にさらに工夫が必要です。

```kotlin
private fun fisherYatesShuffle(bitmap: Bitmap, restore: Boolean) {

    // 念のために処理前のロジックチェックをしています
    val bytesPerPixel = 4 // 1ピクセルあたりのバイト数は4に固定しています
    if (bitmap.byteCount != (bitmap.rowBytes * bitmap.height)) {
        return
    }
    if (bitmap.rowBytes != (bitmap.width * bytesPerPixel)) {
        return
    }

    // ビットマップの画像データ部分を取り出してByteArrayへ変換しています
    val byteBuffer = ByteBuffer.allocate(bitmap.byteCount)
    bitmap.copyPixelsToBuffer(byteBuffer)
    val rawBitmapByteArray = byteBuffer.array()

    // 画像の縦横の分割数を求めます。toInt()はゼロ方向への切り捨て
    val divH: Int = (bitmap.width / divWidthImage).toInt()
    val divV: Int = (bitmap.height / divHeightImage).toInt()

    // 並び替え順情報を生成します
    if (randomSequence == null) {
        randomSequence = IntArray(divH * divV) { it }
        randomSequence?.shuffle()
    }

    // JNIを通してネイティブ関数を呼び出します
    jniFisherYatesShuffle(rawBitmapByteArray, // 画像ビットマップデータ
        bitmap.width, bitmap.height, // 画像の大きさ
        bitmap.rowBytes, // 画像の1行あたりのバイト数
        bytesPerPixel, // 1ピクセルあたりのバイト数
        randomSequence!!, // 並び変え順情報
        divV, divH, // 縦横の分割数
        restore // =false:モザイク処理, =true:復元処理
    )

    // 処理後のByteArrayをビットマップに変換します
    bitmap.copyPixelsFromBuffer(ByteBuffer.wrap(rawBitmapByteArray))
}
```
次にFisherYatesShuffleのネイティブ関数について説明します。
<br>
ほぼほぼ、Wikipediaで紹介されているとおりのFisher-Yates shuffleのロジックそのままです。
swap関数は画像セルのブロック移動処理で、ここがネイティブ化の真骨頂でしょうか。
このあたりはプログラムを読まずにコピペで良いと思います。
<br>
Fisher-Yates shuffleのアルゴリズムについてはWikipediaや、
他の情報がたくさん出てるので、そちらを参照してください。
<br>

```c
static void swap(jint i, jint j, jbyte *rawBitmapByteArray,
                 jint width, jint height,
                 jint rowBytes, jint bytesPerPixel,
                 jint divV, jint divH,
                 jbyte *temp_buffer, jint nbytes) {

    int pi = (i % divH) * (width / divH) +
             (i / divH) * (height / divV) * width;
    int pj = (j % divH) * (width / divH) +
             (j / divH) * (height / divV) * width;

    for (int y = 0;y < (height / divV);y++) {

        int spi = pi * bytesPerPixel + y * rowBytes;
        int spj = pj * bytesPerPixel + y * rowBytes;

        // temp_bufferはブロック移動処理の一時的バッファとして使用
        memcpy(temp_buffer, &rawBitmapByteArray[spi], nbytes);
        memcpy(&rawBitmapByteArray[spi], &rawBitmapByteArray[spj], nbytes);
        memcpy(&rawBitmapByteArray[spj], temp_buffer, nbytes);
    }
}

void FisherYatesShuffle(jbyte *rawBitmapByteArray,
                        jint width, jint height,
                        jint rowBytes, jint bytesPerPixel,
                        const jint *randomSequence,
                        jint divV, jint divH,
                        jboolean restore,
                        jbyte *temp_buffer, jint nbytes) {

    int n = divH * divV;

    if (!restore) {
        // モザイク化の場合はこちら
        for (int i = n-1;i >= 0;i--) {
            int j = randomSequence[i];
            swap(i, j, rawBitmapByteArray,
                 width, height, rowBytes, bytesPerPixel,
                 divV, divH, temp_buffer, nbytes);
        }
    } else {
        // 復元の場合はこちら
        for (int i = 0;i < n;i++) {
            int j = randomSequence[i];
            swap(i, j, rawBitmapByteArray,
                 width, height, rowBytes, bytesPerPixel,
                 divV, divH, temp_buffer, nbytes);
        }
    }
}
```

<!--*****************************************************-->
## 課題
- 小さい画像の場合はモザイク化しても判別されてしまう
  - 1セルあたりの大きさを先に決めてからモザイク処理を行うプログラムなので、
    小さい画像の場合は相対的にセルが大きくなり、判別されやすくなる。
    分割数を決めてからセルの大きさを決めるようにすれば良かった。

- モザイクした画像をリサイズすると復元できない。
  - 1セルあたりの大きさを固定にしているため、モザイク画像をリサイズすると
    分割数が合わなくなり復元できなくなる。
    分割数を固定にしたほうが良かった。

- 並び替え順が(天文学的確率で)順列になった時は画像が判別されてしまう
  - 並び替え順の全部、または一部が等差順列になっていると画像の判別されてしまう。
    もう不運としか言いようがない。

- 並び替え情報を画像とは別に管理(漏洩対策を含む)しなければならない。
  - 宿命的な問題なので改善の余地がない。

<!--*****************************************************-->
## 最後に
画像処理アプリの開発途中で入れたモザイク処理を部分公開しました。
<br>
アルゴリズム自体は簡単ですし、実装もさほど難しくないのですが、
これを画像に応用する手間、高速化のネイティブ化が とても面倒だったので、
同じ苦しみを他の開発者が味わなくても済むように今回紹介しました。
<br>
どうぞコピペで使ってください。

<!--*****************************************************-->

## Reference
[^footnote]: <a href="https://en.wikipedia.org/wiki/Fisher-Yates_shuffle">"Fisher–Yates shuffle" in Wikipedia</a>
[^footnote-2]: 文献２



