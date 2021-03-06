---
layout: post
title: Satellite Track
date: 2021-12-02 10:00:00 +0900
description: satellite tracking
img: 20211202/jaxa1.jpg
categories: [Satellite]
tags: [Satellite]
---
<!--*****************************************************-->
## Overview
近年、ロケットの打ち上げに伴って、超小型人工衛星が放出される事が多くなりました。
超小型人工衛星は、コストが小さく、開発期間も短くて済むことから、
教育機関が宇宙研究目的で開発するようになりました。
少しずつ開発ノウハウが浸透してきたせいで、
学生たちの研究課題としての利用に移ってきたようです。
<br>
先日(2021年11月9日)は、JAXAのイプシロンロケット５号機の打ち上げが成功し、
複数の超小型人工衛星を放出しました。
その１つにKOSEN-1という名称の人工衛星がありますが、
これは全国の国立高専10校が連携して開発したものだそうです。

驚くことに、この人工衛星に搭載されているコンピュータはRaspberry Pi Zeroを改造したもので、
数百円〜数千円程度で簡単に手に入る民生品です。
筆者が半導体会社に入社した当時、
JAXAの前身である宇宙開発事業団がH-Ⅰ,H-Ⅱロケットを打ち上げていた時期でした。
ロケットや人工衛星に搭載される半導体は、宇宙線に耐性のある特別品でなければならず、
民生品の半導体のマニュアルに「宇宙では使わないでください」と、わざわざ注意書きを入れていたものです。

話が脱線しましたが、
高専の学生さん達の思いが詰まったKOSEN-1が発信するCWビーコンを受信してみたくなり、
手持ちのRTL-SDR+プリアンプを利用した受信装置を考えたのですが、
RTL-SDRの性能や、北側の受信が出来ない集合住宅、アンテナを作る手間、等の課題が多いため断念し、
WebSDRを利用することにしました。

WebSDR (http://www.websdr.org/)とは、
世界各地のボランティアがSDR (Software-Defined Radio)サーバーを設置し、
インターネット経由で誰でもコントロール、電波を受信できるサービスのことです。
現在では180サーバーが公開されており、
人工衛星が利用するアマチュア無線帯域(145MHz帯,430MHz帯)を受信することができます。

今回は、KOSEN-1を含めた他の超小型人工衛星のCWビーコンも受信するために、
AMSAT (https://www.amsat.org/status/)から運用中と思われるものをピックアップした人工衛星のリスト (satellite.txt)と、
SDRサーバーの観測地のリスト (websdr.txt)から、
これから８時間以内に受信可能となる人工衛星と観測地とその時刻を求めるPythonプログラムを作成しました。

<!--*****************************************************-->
## Source Code
Source code may be found as the Python3 program in the GitHub repository.

[github.com/HideoMiyauchi/SatelliteTrack](https://github.com/HideoMiyauchi/SatelliteTrack)

<!--*****************************************************-->
## Usage
Very easy to get it working.

```
$ python satellite_track.py
```

<!--*****************************************************-->
## 実行結果
このプログラムを実行すると、受信できる人工衛星と観測地サーバーと時刻のリストが表示されます。
<br>
リストの見方について説明します。
<br>
```
(省略)
----------------------
CAS-6 (TO-108), 145.91MHz
Websdr Receiver operated by SV1RVL in Athens Greece
http://sv1rvl.ddns.net:8901/
21/12/02 18:36:09, 40.0, 11.7, 912.5
21/12/02 18:37:41, 88.7, 284.2, 621.3
21/12/02 18:39:12, 40.0, 194.5, 911.3
----------------------
NAYIF-1 (EO-88), 145.94MHz
WebSDR-DK4KM in Hattorf / Germany
(省略)
```
まず、観測地の上空を通過する衛星名とビーコン周波数です。
> CAS-6 (TO-108), 145.91MHz

CAS-6は2019年12月に打ち上げられた中華人民共和国の小型人工衛星で、
490mmx499mmx430mmの立方体に太陽電池を搭載し、質量は約35kgです。
CWテレメトリビーコン周波数145.91MHzになっています。
<br>
![cas6photo](/assets/img/20211202/tianqin-1-cas-6__2.jpg){: width="300" }
<br>
<br>
次に、観測地の名前とSDRサーバーへのアクセスURLです。
ギリシャのアテネが観測地です。
<br>
> Websdr Receiver operated by SV1RVL in Athens Greece  
  http://sv1rvl.ddns.net:8901/

そして上から、見え始め、最高高度、見え終わり、左から、日付、時刻、高度、方位角、距離(km)です。
<br>
> 21/12/02 18:36:09, 40.0, 11.7, 912.5  
21/12/02 18:37:41, 88.7, 284.2, 621.3  
21/12/02 18:39:12, 40.0, 194.5, 911.3

なお、この時刻はJSTです。
12月2日18時36分にサーバー (http://sv1rvl.ddns.net:8901/)へブラウザからアクセスし、
SDRの電波形式をCWに、受信周波数を145.91MHzに設定すれば、
18時39分頃までモールス信号を受信できます。
ドップラーシフトがあるので145.91MHz周辺を調べていれば、
それらしい信号を捉えることができます。
<br>
なお、モールス信号が解らない場合でも、
モールス信号を解析するスマホの無料アプリがあるので、
事前にそれを準備しておきます。

<!--*****************************************************-->
## ファイルの説明
- satellite_track.py
<br>
本プログラムは軌道計算のために、NORADのTLEをダウンロードします。
ビーコン受信の条件として、
  - 条件1. 人工衛星の最高高度が80度以上
  - 条件2. 人工衛星に陽が当たっている
  - 条件3. 観測地の太陽高度が10度以上の日中

  を考慮しています。
条件2については太陽電池が発電中の時にビーコン送信パワーが大きいはずです(違いますか？)。
条件3については気象衛星NOAAの画像受信の時の為です(必要無ければ条件から外してください)。
- websdr.txt
<br>
WebSDR (http://www.websdr.org/)に記載されたSDRサーバーをリストにしたものです。
３行目はGrid Square Locatorです。
<br>
- satellite.txt
<br>
AMSAT (https://www.amsat.org/status/)から運用中と思われる人工衛星と、そのビーコン送信周波数をリストにしています。さらに気象衛星NOAAと国際宇宙ステーションを追加しました。

<!--*****************************************************-->
## KOSEN-1の受信方法
今回の目的である国立高専が開発した超小型衛星KOSEN-1は、100mmx100mmx200mm 2kgの大きさで、
腕で抱えられるぐらい小さいものです。
<br>
![kosen1photo](/assets/img/20211202/d31b164cea2b3eea057175dea6168e1f.jpg){: width="430" }
<br>
KOSEN-1の軌道要素はNORADのTLEには未だ登録されていないので、
本プログラム中に次のように入れてやる必要があります。
なお、KOSEN-1の軌道要素については最新のものを(ネットで探して)使うようにしてください。
```
(省略)
# NORADからTLEデータをダウンロードする
tle_data = {}
for tle_url in tle_urls:
    tle_data.update(load.tle(tle_url))

# add KOSEN-1
kosen_1_name = 'KOSEN-1'
satellites.append([kosen_1_name, 435.525])
kosen_1_tle = EarthSatellite(
    '1 49402U 21102H   21336.20003684  .00001637  00000-0  12605-3 0  9997',
    '2 49402  97.5584  35.1773 0022887 169.2720 276.0408 15.02837324  3475',
    kosen_1_name, ts)
tle_data.update({kosen_1_name:kosen_1_tle})

# 人工衛星のビーコン周波数を受信できる観測地を調べる
iter = []
for satellite in satellites:
    satellite_freq = satellite[1]
(省略)

```
このプログラムで計算した時刻 (2021年11月13日16時28分 JST)に、南アフリカのヨハネスブルグで上空を通過したKOSEN-1からのCWビーコンを受信することができました。
そのときのSDRの画面キャプチャをYouTubeにアップしました。

<iframe src="https://www.youtube.com/embed/QakQaCBg6QA" title="YouTube video player" frameborder="10" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<!--*****************************************************-->
## 最後に
私にとっての宇宙とは、宇宙開発事業団が独占する専門領域だったのですが、
それが今や、学生さんの人工衛星が宇宙を飛ぶ時代になったのは感慨深いものがあります。
このKOSEN-1を開発した学生さん達が将来にJAXAやNASAで活躍し、
我々に宇宙のロマンを届けてくれることをとても期待しています。

<!--*****************************************************-->
## 謝辞
写真を使わせて頂きました。プログラム開発の参考にさせて頂きました。
ありがとうございました。

- [(C) 宇宙航空研究開発機構（JAXA）](https://www.jaxa.jp/)
- [大阪市立科学館](http://www.sci-museum.kita.osaka.jp/~egoshi/astronomy/python/python_iss_skyfield.html)



