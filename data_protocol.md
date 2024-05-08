# 学ロボ2024 CAN/Serialプロトコル  

基本的にIDとデータの意味は一対一対応する構成となっている。  

## IDの構成について

IDはPriority level,Data type,Board ID,Register IDの4つの情報を組み合わせで表現される。  

#### Priority level

優先度。  
ただし優先的に送信されるようなコードが構築されることを保証するものではなく、0が優先されるというCANバスの特性を利用する簡易的な優先度である。  

#### Data type

| 名称 | ID |
|:--:|:--:|
| 共通データ形式（強制受信） | 0xF |
| 共通データ形式 | 0x0 |
| 電源基板用データ | 0x1 |
| ロボマス制御用データ | 0x2 |
| 汎用GPIO用データ | 0x3 |

Data type=0xFの場合はboard_idの一致不一致関わらず受信し処理する。  
非常停止コマンドなど、バス上の全デバイスが受信すべきものに使用することを想定。  

#### Board ID

ロータリーDIPなどで指定される基板固有の値。  

#### Register ID

データの具体的な内容を意味する。  
詳細は下参照

### CANの場合  

拡張IDを用いる。  

| bit | 名称 |
|:--:|:--:|
|28|予約済み 0推奨|
|27:24|Priority level|
|23:20|Data type|
|19:16|Board ID|
|15:0|Register ID|

### マイコン-PC間のUSB通信

基本的にSLCANを使用

https://www.canusb.com/docs/can232_v3.pdf

↑プロトコルはこの辺参照

たとえば拡張ID=0x12345678,DLC=8,Data={0x00,0x11,0x22,0x33,x044,0x55,0x66,0x77}を送りたいなら  

```
T1234567880011223344556677[CR]
```

をシリアルで送ってあげればよい

[python-can](https://python-can.readthedocs.io/en/stable/)が使える  

[python-canでのslcan](https://python-can.readthedocs.io/en/stable/interfaces/slcan.html)  

### ~~Serialの場合~~非推奨オレオレプロトコル  

どうしてもUSB側の帯域が足りなかった場合復活するかも  

<details><summary>詳細</summary><div>

|byte| bit | 名称 | 概要 |
|:--:|:--:|:--:|:--:|
|0|3|r/w|リモートフレーム枠|
|0|2:0|priority level||
|1|7:4|Data type|データの種類|
|1|3:0|Board ID|基板の固有ID|
|2|7:0|Register ID 15~8||
|3|7:0|Register ID 7~0||

0byte目はパケットの情報、1-3byte目はデータの情報的な

実際にはCOBS形式にエンコードしてやり取りする

</div></details>

## register IDの詳細

### 共通データ形式（Data type=0x0,0xF）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0000|NOP||||
|0x0001|ID_RQ|uint8_t|-|これを受信したデバイスは自身のIDを返せ|
|0x0002|~~SAVE_PARAM~~|-|-|いつかやる|
|0x000E|EMS|-|-|非常停止|
|0x000F|RESET_EMS|-|-|非常停止解除|

#### ID_RQ

リモートフレームでID_RQを送信することでバス上のデバイスを調べることができる。  
リクエストを受け取ったデバイスは、Board IDに自身のID（ロータリーDIPの値）、data[0]に基板の形式を入れてData type 0x0で返す。  

基板形式は以下の通り。  
| 名称 | ID |
|:--:|:--:|
| 電源基板 | 0x1 |
| ロボマス制御基板 | 0x2 |
| 汎用GPIO基板 | 0x3 |

### 電源基板（Data type=0x1）

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0000|NOP||||
|0x0001|PCU_STATE|uint8_t|r|電源の状態|
|0x0002|CELL_N|uint8_t|r/w|リポのセル数|
|0x0003|EX_EMS_TRG|uint8_t|r/w|自動非常停止の条件設定|
|0x0004|EMS_RQ|bool|w|遠隔非常停止リクエスト|
|0x0005|COMMON_EMS_EN|bool|r/w|非常停止がかかった時にCommonReg::EMERGENCY_STOPを発報するか 初期値true|
|0x0010|OUT_V|float(V)|r|現在の電圧|
|0x0012|V_LIMIT_HIGH|float(V)|r/w|セル当たりの電圧がこれを超えるとアラート|
|0x0013|V_LIMIT_LOW|float(V)|r/w|セル当たりの電圧がこれを下回るととアラート|
|0x0020|OUT_I|float(A)|r|現在の電流|
|0x0022|I_LIMIT|float(A)|r/w|この電流を超えるとアラート|
|0x00F0|MONITOR_PERIOD|uint16_t(ms)|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0x00F1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定|

#### PCU_STATE  

**PCU_STATE!=0ならなんかヤバいことが起きてるので非常停止シーケンス起動しろ**

|bit|名称|概要|
|:--:|:--:|:--:|
|0|EMS|非常停止が入っているか|
|1|SOFT_EMS|ソフトウェア非常停止がかかっているか|
|2|OVA|過電圧アラート|
|3|UVA|低電圧アラート|
|4|OIA|過電流アラート|

なおソフトウェア非常停止は遠隔非常停止や過電圧・過電流非常停止など、ソフトウェアによる非常停止全般を指す。  
EMS bitは電源基板のリレーのON/OFFを示す。  
よって非常停止スイッチによる非常停止・ソフトウェア非常停止のどちらの場合にも1となる。  

PCU_STATEは変化があった場合即座に発報される。  

#### EX_EMS_TRG

各異常が発生した際にソフトウェア非常停止を入れるかの選択

|bit|名称|概要|
|:--:|:--:|:--:|
|0:1|-|予約済み|
|2|OVA_EMS_EN|過電圧アラート時に非常停止|
|3|UVA_EMS_EN|低電圧アラート時に非常停止|
|4|OIA_EMS_EN|過電流アラート時に非常停止|

初期値は0で、1で有効

### ロボマス制御基板データ（Data type=0x2）  

nはモーターID（0~3）  
cは全モーターに共通する設定項目（例えば0x105も0x205も結果的に同じものが書き換わるのでどちらでも良い）

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0n00|NOP||||
|0x0n01|MOTOR_TYPE|uint8_t|r/w|接続するドライバの種類|
|0x0n02|CONTROL_TYPE|uint8_t|r/w|enum CONTROL_TYPE|
|0x0n03|GEAR_RATIO|float|r/w|モーターのギア比|
|0x0n04|MOTOR_STATE|bool|r|ドライバがつながっていたらtrue|
|0x0c05|CAN_TIMEOUT|uint16_t(ms)|r/w|この時間CAN信号が送られてこなければ停止 0で無効化|
|0x0n10|PWM|float|r|現在のPWM|
|0x0n11|PWM_TARGET|float|r/w|PWM指令値|
|0x0n20|SPD|float(rad/s)|r|現在の速度|
|0x0n21|SPD_TARGET|float(rad/s)|r/w|目標速度|
|0x0n22|PWM_LIM|float|r/w|トルク制限|
|0x0n23|SPD_GAIN_P|float|r/w|速度Pゲイン|
|0x0n24|SPD_GAIN_I|float|r/w|速度Iゲイン|
|0x0n25|SPD_GAIN_D|float|r/w|速度Dゲイン|
|0x0n30|POS|float(rad)|r/w|現在の位置|
|0x0n31|POS_TARGET|float(rad)|r/w|目標位置|
|0x0n32|SPD_LIM|float(rad/s)|r/w|速度制限|
|0x0n33|POS_GAIN_P|float|r/w|位置Pゲイン|
|0x0n34|POS_GAIN_I|float|r/w|位置Iゲイン|
|0x0n35|POS_GAIN_D|float|r/w|位置Dゲイン|
|0x0n36|ABS_POS|float(rad)|r|アブソリュートエンコーダによる絶対位置|
|0x0n37|ABS_SPD|float(rad/s)|r|アブソリュートエンコーダによる絶対速度|
|0x0n38|ABS_ENC_INV|bool|r/w|アブソエンコーダの回転方向反転|
|0x0n39|ABS_TURN_CNT|int32_t|r/w|アブソエンコーダの回転回数|
|0x0cF0|MONITOR_PERIOD|uint16_t(ms)|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0x0nF1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定|

#### POS

現在の角度を表している。
上書き可能であり、例えば現在位置を原点としたければ0を書き込むと現在位置を基準とした位置制御が可能になる。  

#### MOTOR_TYPE

||名称|
|:--:|:--:|
|0x0|C610/C620|
|0x1|VESC|

VESCにしたときは必ずCONTROL_TYPEをPWM_MODEにすること。  
ABS_POSITION_MODEはワンチャン動作するかもしれないが非推奨  

#### GEAR_RATIO

初期値は36(M2006)となっている
M3508を使いたいなら19にすると出力軸角度とPOSが一致する。

#### CONTROL_MODE

||名称|備考|
|:--:|:--:|:--:|
|0x0|PWM_MODE|オープンループ|
|0x1|SPEED_MODE|速度制御|
|0x2|POSITION_MODE|位置制御|
|0x3|ABS_POSITION_MODE|絶対位置制御（AS5600利用）|

#### CAN_TIMEOUT

ここで設定した時間CANが受信できなければ自動的に非常停止状態に移行する

#### ABS_POS/ABS_SPD

アブソリュートエンコーダエンコーダによる位置・速度情報  
回転速度が速すぎると回転数カウントがバグって誤差が出る可能性があるので注意（よっぽど大丈夫だけど）
POSのようには上書きできない  

#### 非常停止

共通コマンドのEMERGENCY_STOPが来るとすべてのモータはPWMモードに移行して停止する。  
RESET_EMERGENCY_STOPでモード設定などは復活するが、電源が落ちた状態でモーターを回すと、回転数情報などがバグるためPOSがずれる可能性が高いため注意。  
非常停止解除時に原点出しシーケンスを起動するのが望ましい。  

### GPIO基板（Data type = 0x3）

各コネクタをピン、ピンの集合をポートと呼称することとする。  

nは操作したいピン番号（銀将の場合0~8）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0000|NOP||||
|0x0001|PORT_MODE|uint16_t|w|各ピンのモード一括設定|
|0x0002|PORT_READ|uint16_t|r|各ピンの現在の状態を一括で読む|
|0x0003|PORT_WRITE|uint16_t|w|OUTPUTモードにしたPINの出力 PICで言うLAT|
|0x0004|PORT_INT_EN|uint16_t|w|ピン変化割り込みの有効化|
|0x0005|ESC_MODE_EN|uint16_t|w|サーボモードで駆動するピンの選択|
|0x001n|PWM_PERIOD|uint16_t|r/w|ソフトウェアPWMの周期|
|0x002n|PWM_DUTY|uint16_t|r/w|ソフトウェアPWMのduty|
|0x00F0|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0x00F1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~0x3F|

PORT_MODEは0で出力、1で入力モード。

各ピンは出力モードにした時、ソフトウェアPWMとして駆動することができる。  

#### 入力モードの際の動作

基本的に各ピンは入力モードでマイコン内蔵抵抗によりプルアップされている。そのため何もセンサー等を接続しない場合、PORT_READを読み取ると0x1FFといった値が帰ってくる。  
なお、PORT_READレジスタはHIGHで1、LOWで0となる。

#### ソフトウェアPWMの動作

PWM出力に用いるカウンタは50kHzでカウントアップしている。  
たとえば50Hz、duty比30%（サーボ用PWM）を出力する場合、  

```math
\text{PWM\_PERIOD}=\frac{1}{50[Hz]}/\frac{1}{50000[Hz]}=1000
```

```math
\text{PWM\_DUTY}=\text{PWM\_PERIOD}*0.3=300
```

と設定すればよい。  
ピンに対応するPORT_WRITEレジスタのビットが1の時PWM出力され、0の時は出力されず、LOWとなる。  
PWMを出力せず単にGPIOとして動作させたい場合はDuty比100%となるようPWM_DUTYを設定（0xFFFFなど）すれば良い。  

#### ピン変化割り込み動作

ピンの状態に変化があった際に即座に信号を発報して欲しい場合、PORT_INT_ENを用いることができる。  
変化を通知して欲しいポート番号に対応するビットを1とすることで有効化、0で無効化となる。  
なおチャタリング除去処理により0.5ms程度の遅れが生じる可能性はあるため注意。  

#### ESC_MODE_EN

ラジコン用ESCを駆動するためのPWMを生成するやつ。  
1で有効になり、設定されたportは自動的に50HzのPWM出力モードに切り替わる。  
PWM_DUTYに**50~100**の値を入力することで制御できる。  
なんかキモイ値だがこれが慣例といわれるやつであるので許せ。  

ESCモードを有効にした際と非常停止解除信号(0x0pF0000F)を受け取った際に自動的に起動シーケンス（10秒程度）が起動するため、この間はポートは使用できない。  

すでにESC_MODEとなっている状態でESC_MODE_ENに有効化指令を送っても起動シーケンスは始動しない。  
始動したい場合は一度無効化してから再度有効化するなどすればよい。  

## MONITOR_PERIOD/REGについて（全データ形式共通）  

MONITOR_PERIODで設定した周期（ms単位）でフィードバックするデータを選択する。  

なお周期は基板内ですべて共通なので個別に設定することはできない（ID0の基板のモータ1とモータ２のフィードバック周期を個別に指定する、など）。

MONITOR_REGで設定したデータが定期的に送信される。  
たとえば、ロボマス制御基板において現在の速度を定期的にフィードバックしてほしければMONITOR_REGの0x20ビット目を1にして送信すればよい。  

## データのサイズなど

bool型は1byte。0でfalseでそれ以外はtrue。  
データはすべてリトルエディアン。  