# 学ロボ2024 CAN/Serialプロトコル  

### CANの場合  

| bit | 名称 | 概要 |
|:--:|:--:|:--:|
|28|予約済み|ただしCANの物理プロトコルの特性上本質的に優先度　0推奨|
|27:24|Priority level|優先度（0~Fで0が最高）|
|23:20|Data type|データの種類|
|19:16|Board ID|基板の固有ID(0~F)|
|15:0|Register ID||

なお、Data typeは次の様
| 名称 | ID |
|:--:|:--:|
| 共通データ形式（強制受信） | 0xF |
| 共通データ形式 | 0x0 |
| 電源基板用データ | 0x1 |
| ロボマス制御用データ | 0x2 |
| 汎用GPIO用データ | 0x3 |

Board IDはロータリーDIPなどで指定される値。  
ID=0xFの場合はboard_idの一致不一致関わらず受信し処理する。

### マイコン-PC間のUSB通信

基本的にSLCANを使用

https://www.canusb.com/docs/can232_v3.pdf

↑プロトコルはこの辺参照

たとえば拡張ID=0x12345678,DLC=8,Data={0x00,0x11,0x22,0x33,x044,0x55,0x66,0x77}を送りたいなら  
```
T1234567880011223344556677
```
をシリアルで送ってあげればよい

[python-can](https://python-can.readthedocs.io/en/stable/)が使える  

[python-canでのslcan](https://python-can.readthedocs.io/en/stable/interfaces/slcan.html)  

### ~~Serialの場合~~非推奨オレオレプロトコル  

|byte| bit | 名称 | 概要 |
|:--:|:--:|:--:|:--:|
|0|3|r/w|リモートフレーム枠|
|0|2:0|priority level||
|1|7:4|Data type|データの種類|
|1|3:0|Board ID|基板の固有ID|
|2|7:0|Register ID 15~8||
|3|7:0|Register ID 7~0||

~~0byte目はパケットの情報、1-3byte目はデータの情報的な~~

~~実際にはCOBS形式にエンコードしてやり取りする~~

どうしてもUSB側の帯域が足りなかった場合復活するかも

## 共通データ形式（Data type=0x0,0xF）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0|REQEST_ID|-|-|これを受信したデバイスは自身のIDを返せ|
|0x1|RESPONSE_ID|uint8_t|-|データ部分は基板形式 下参照|
|0x2|SAVE_PARAM|-|-|ゲイン等現在のパラメータを保存|
|0xE|EMERGENCY_STOP|-|-|非常停止|
|0xF|RESET_EMERGENCY_STOP|-|-|非常停止解除|

REQEST_IDに対して返答する際は、Board IDに自身のID（ロータリーDIPの値）、data[0]に基板の形式を入れてData type 0x0で返す。
基板形式は以下の通り。  
| 名称 | ID |
|:--:|:--:|
| 電源基板 | 0x1 |
| ロボマス制御基板 | 0x2 |
| 汎用GPIO基板 | 0x3 |

## ロボマス制御基板データ（Data type=0x2）  

nはモーターID（0~3）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0xn00|NOP|||
|0xn01|MOTOR_TYPE|uint8_t|r/w|enum MOTOR_TYPE|
|0xn02|CONTROL_TYPE|uint8_t|r/w|enum CONTROL_TYPE|
|0xn03|GEAR_RATIO|float|r/w|モーターのギア比|
|0xn04|MOTOR_STATE|uint8_t|r|現在のモーターの状態表示|
|0xn10|PWM|float(-1~1)|r|現在のPWM|
|0xn11|PWM_TARGET|float(-1~1)|w/r|
|0xn20|SPD|float(rad/s)|r|現在の速度|
|0xn21|SPD_TARGET|float(rad/s)|r/w|目標速度|
|0xn22|PWM_LIM|float(rad/s)|r/w|トルク制限|
|0xn23|SPD_GAIN_P|float|r/w|速度Pゲイン|
|0xn24|SPD_GAIN_I|float|r/w|速度Iゲイン|
|0xn25|SPD_GAIN_D|float|r/w|速度Dゲイン|
|0xn30|POS|float(rad/s)|r/w|現在の位置|
|0xn31|POS_TARGET|float(rad/s)|r/w|目標位置|
|0xn32|SPD_LIM|float(rad/s)|r/w|速度制限|
|0xn33|POS_GAIN_P|float|r/w|位置Pゲイン|
|0xn34|POS_GAIN_I|float|r/w|位置Iゲイン|
|0xn35|POS_GAIN_D|float|r/w|位置Dゲイン|
|0xnF0|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0xnF1|MONITOR_REG1|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~0x39|
|0xnF2|MONITOR_REG2|uint64_t|r/w|モニターするレジスタを設定 reg ID 0x40~0x79|
|0xnF3|MONITOR_REG3|uint64_t|r/w|モニターするレジスタを設定 reg ID 0x80~0xB9|
|0xnF4|MONITOR_REG4|uint64_t|r/w|モニターするレジスタを設定 reg ID 0xC0~0xE9|

現在位置は上書き可能。  
たとえばPOSに0を書きこめば現在位置が原点となる  

### MOTOR_TYPE

PWM周りの設定が若干変わるだけなので現状ほとんど意味はない

||名称|備考|
|:--:|:--:|:--:|
|0x0|M2006||
|0x1|M3508||

### GEAR_RATIO

初期値は36(M2006)となっている
M3508を使いたいなら19にすると出力軸角度とPOSが一致する。

### CONTROL_MODE

||名称|備考|
|:--:|:--:|:--:|
|0x0|PWM_MODE|オープンループ|
|0x1|SPEED_MODE|速度制御|
|0x2|POSITION_MODE|位置制御|
|0x3|ABS_POSITION_MODE|絶対位置制御（AS5600利用）|

なおABS_POSITION_MODEは未実装  

### MONITOR_REG

MONITOR_PERIODで設定した周期でフィードバックするデータを選択する。  

たとえば現在の速度を定期的に送ってほしければMONITOR_REG1の0x20ビット目を1にして書きこむ。  
