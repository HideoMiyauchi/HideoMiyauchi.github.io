---
layout: post
title: Flight Simulation
date: 2021-04-09 10:00:00 +0900
description: Flight Dynamics
img: 20210405/flight.jpg
categories: [Simulation, Flight]
tags: [FlightSimulation]
---
<!--*****************************************************-->
## Overview
飛行力学の専門書[^footnote]を参考に、長距離戦略爆撃機(XB-70A)[^fn-nth-2]の運動方程式を求め、
オイラー法による数値解法で、操舵制御による飛行経路を3D表示するPythonプログラムを作成しました。

<!--*****************************************************-->
## Source Code
Source code may be found as the Python3 program in the GitHub repository.

<https://github.com/HideoMiyauchi/FlightSimulation>

<!--*****************************************************-->
## Usage
Very easy to get it working.

```
$ python flight_demo.py
```

<!--*****************************************************-->
## 座標系と記号の定義
座標系と記号を定義します。
<br>
機体に固定した座標系の機首前方をx軸、右翼方向をy軸、下側をz軸とし、それぞれの軸の回転をロール、ピッチ、ヨーと呼びます。
ロール角度は $$\phi$$、ロール角速度はp、ピッチ角度は$$\theta$$、ピッチ角速度はq、ヨー角度は$$\psi$$、ヨー角速度はrとします(図(a))。
<br>
機体速度V、仰角$$\alpha$$および横滑り角$$\beta$$で飛行する場合のX軸、Y軸、Z軸の速度成分を{u,v,w}とします(図(b))。
<br>
舵は昇降舵(elevator)、方向舵(rudder)、補助翼(aileron)、フラップ(flap)で、それぞれの角度を{$$\delta_e, \delta_r, \delta_a, \delta_f$$}で表します(図(c))。
<br>
![bodyaxes](/assets/img/20210405/bodyaxes.jpg){: width="400"}

<!--*****************************************************-->
## 長距離戦略爆撃機 XB-70A とは
このプログラムで用いた爆撃機(XB-70A)について紹介します。
<br>
アメリカ空軍の「ヴァルキリー計画」に基づき、ノースアメリカン社が開発した戦略爆撃機である。最高速度マッハ3で、米国領アラスカとソビエト連邦首都モスクワの間を無着陸で往復可能な超音速戦略核爆撃機として計画されました(wikipediaより)
<br>
![xb70photo](/assets/img/20210405/XB70photo.jpg){: width="400" .right}
<br>
"The XB-70A was originally designed as a weapons systems with long range supersonic cruise capabilities. The two aircraft build became research aircraft to explore SST-related programs.
The two XB-70A's were identical except that the first airplane (XB-70A-1) had zero geometric dihedral while the second had 5 deg geometric dihedral.
The first airplane is considered here.
Pitch control employs interconnected elevon and canard surfaces except in takeoff and landing where the canard is locked and a fixed canard flap is used. Roll control is obtained through differential action of the elevons. Yaw control is provided by rotation of the vertical stabilizers about a 45 deg hinge line.
The airplane is equipped with stability augmentation in all axes.
Data shown here is a composite of many sources. The object was to use flight test data where possible."
<br>
![Destop View](/assets/img/20210405/XB70Abody.jpg){: width="400" .right}

<!--*****************************************************-->
## 有次元微係数への変換
文献[^fn-nth-2]に記載されたXB-70AのNondimensional Stability Derivatives(無次元微係数)は、機体の大きさや速度に拘わらず機体の形状のみに対する空気力の大きさを表したもので、これを動圧や翼面積・空気密度を考慮したDimensional Stability Derivatives(有次元微係数)に変換します。
その変換方法は文献[^footnote]を参考にしています。同時に、長さ・重さ・度数の単位変換も必要です。

\$\$
\begin{CD}
無次元微係数\cr \\
@VV{単位変換・有次元化}V\cr \\
有次元微係数(X_uなど)
\end{CD}
\$\$

<!--*****************************************************-->
## 微小撹乱運動方程式

空間上の運動は、3軸方向の運動と3軸回りの回転運動の6自由度運動(6個の状態変数)と、空間上の姿勢を決める3個の関係式の合計9個の微分方程式で表されます。航空機は左右対称な形状であることから、運動が微小に変化する場合には、縦と横方向の運動に分けて扱うことができます。

縦方向
: 縦方向の運動方程式は次のように表されます。
なお、フラップ項を削除し、スロットル項を追加しています。
縦方向の状態変数は $$\{\ u,\ \alpha,\ q,\ \theta,\ \delta_e,\ \delta_t\ \}$$ です。

\$\$ \dot u = X_u\ u + X_\alpha\ \alpha - \frac{g\ \cos \theta_0}{57.3}\theta + {X_\delta}_t\ \delta_t \$\$

\$\$ \dot \alpha = \bar Z_u\ u + \bar Z_\alpha\ \alpha + q - \frac{g\ \sin \theta_0}{57.3}\theta + {\bar Z_\delta}_e\ \delta_e \$\$

\$\$ \dot q = M'_u\ u + M'\_\alpha\ \alpha + M'_q\ q +M’\_\theta\ \theta + {M'\_\delta}_e\ \delta_e \$\$

\$\$ \dot \theta = q \$\$

横方向
: 横方向の運動方程式は次のように表されます。
ラダー項は削除しています。ラダーはロール動作の際に横滑りを減らす重要な働きをするのですが、本プログラムでは横滑りは考慮していません。
横方向での状態変数は $$\{\ \beta,\ p,\ r,\ \phi,\ \delta_a\ \}$$ です。

\$\$ \dot \beta = \bar Y_\beta\ \beta + \frac{\alpha_0}{57.3}\ p - r + \frac{g\ \cos \theta_0}{V}\ \phi \$\$

\$\$ \dot p = L'\_\beta\ \beta + L'_p\ p + L'_r\ r + {L'\_\delta}_a\ \delta_a \$\$

\$\$ \dot r = N'\_\beta\ \beta + N'_p\ p + N'_r\ r + {N'\_\delta}_a\ \delta_a \$\$

\$\$ \dot \phi = p + r\ \tan \theta_0 \$\$

初期値は、高度、速度、仰角を次のように設定します。

\$\$ 高度 h_0 = 1500\ (feet) \$\$

\$\$ 速度 V_0 = 100 \$\$

\$\$ 仰角 \theta_0 = 7.5\ (deg) \$\$

<!--*****************************************************-->
### 地球座標への変換
機体座標系での移動速度を地球座標系で表すためのオイラー角の回転を施します。
機体座標系での移動速度$$ (u,v,w) $$ は、仰角$$\alpha$$、横すべり角$$\beta$$ で、

\$\$u = (V + V_0) * \cos \beta \ \cos \alpha\$\$

\$\$v = (V + V_0) * \sin \beta\$\$

\$\$w = (V + V_0) * \cos \beta \ \sin \alpha\$\$

このように表すことができ、
回転の変換行列は次のようになります。

\$\$ a11 = \cos \theta \ \cos \psi \$\$
\$\$ a12 = \sin \phi \ \sin \theta \ \cos \psi - \cos \phi \ \sin \psi \$\$
\$\$ a13 = \cos \phi \ \sin \theta \ \cos \psi + \sin \phi \ \sin \psi \$\$
\$\$ a21 = \cos \theta \ \sin \psi \$\$
\$\$ a22 = \sin \phi \ \sin \theta \ \sin \psi + \cos \phi \ \cos \psi \$\$
\$\$ a23 = \cos \phi \ \sin \theta \ \sin \psi - \sin \phi \ \cos \psi \$\$
\$\$ a31 = \sin \theta \$\$
\$\$ a32 = \sin \phi \ \cos \theta \$\$
\$\$ a33 = \cos \phi \ \cos \theta \$\$

\$\$
\left\lceil\matrix{
\dot X_E\cr
\dot Y_E\cr
\dot Z_E
}\right\rceil
 = \left\lceil\matrix{
a11, & a12, & a13\cr
a21, & a22, & a23\cr
a31, & a32, & a33
}\right\rceil
\left\lceil\matrix{
u\cr
v\cr
w
}\right\rceil
\$\$

<!--*****************************************************-->
## 飛行制御
状態変数フィードバックによる飛行制御を行います。
ピッチ角、ロール角のコマンドを与え、それを昇降舵(elevator)、補助翼(aileron)の操舵角の入力とします。
本プログラムは飛行経路を求めるだけのプログラムなので、長周期振動や短周期振動については考慮していません。
<br>
![image tooltip here](/assets/img/20210405/autoPilot.jpg){: width="600" }

<!--*****************************************************-->
## シミュレーション

飛行シナリオは、飛行開始から30秒までは、ピッチ角30[deg]、ロール角20[deg]、30秒〜130秒までは、ピッチ角-10[deg]、ロール角50[deg]のコマンド指示を与えることにします。ここは好きなシナリオを設定してください。
<br>
表示する機体は下手なワイヤーフレームなので分かりにくいのですが、尾翼位置が赤いライン、車輪位置が青いラインのつもりです。
<br>
![Destop View](/assets/img/20210405/figure2.jpg){: width="200" .right}
![Destop View](/assets/img/20210405/FlightMove.gif){: width="500" .right}

YouTube(Android) <https://youtu.be/zlw8Ba14BAs>

YouTube(Android) <https://youtu.be/eNX422cPve4>


<!--*****************************************************-->
## 最後に
映画「インデペンデンス・デイ」のラストで爆弾ごと突っ込む老パイロットの操縦を見ると
操縦桿１本で簡単に操縦できそうに思えましたが、
コンピュータ制御無しの機体は横滑りの修正、長・短周期振動の抑制などの複雑な操舵が必要で
あることが分かりました。
<br>
大きな旅客機で安心・快適に旅行できるのもコンピューターのおかげです。

<!--*****************************************************-->
## Reference
![image tooltip here](/assets/img/20210405/069089cvr.jpg){: width="150" .right }
<br>

[^footnote]: 片柳亮二.『航空機の飛行力学と制御(POD版)』.森北出版株式会社,2017

[^fn-nth-2]: Robert K H, Wayne F J.Aircraft handling qualities data.NASA CR-2144,1972.






