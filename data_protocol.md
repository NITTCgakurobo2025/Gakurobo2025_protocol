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
| ~~共通データ形式（強制受信）~~ | 0xF |
| ~~共通データ形式~~ | 0x0 |
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

## ~~共通データ形式（Data type=0x0,0xF）~~ 一旦開発停止  

めんどくさくなってきたので取敢えず凍結します

<details><summary>概念</summary><div>

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

</div></details>

## 電源基板（Data type=0x1）

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x00|NOP||||
|0x01|PCU_STATE|uint8_t|r|電源の状態|
|0x02|CELL_N|uint8_t|r/w|リポのセル数|
|0x03|EX_EMS_TRG|uint8_t|r/w|自動非常停止の条件設定|
|0x04|EMS_RQ|bool|r/w|遠隔非常停止リクエスト|
|0x10|OUT_V|float|r|現在の電圧|
|0x12|V_LIMIT_HIGH|float|r/w|セル当たりの電圧がこれを超えるとアラート|
|0x13|V_LIMIT_LOW|float|r/w|セル当たりの電圧がこれを下回るととアラート|
|0x20|OUT_I|float|r|現在の電流|
|0x22|I_LIMIT|float|r/w|この電流を超えるとアラート|
|0xF0|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0xF1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~0x3F|

### PCU_STATE  

**PCU_STATE!=0ならなんかヤバいことが起きてるので非常停止シーケンス起動しろ**

|bit|名称|概要|
|:--:|:--:|:--:|
|0|EMS|非常停止スイッチON|
|1|EX_EMS|基板内非常停止ON|
|2|OVA|過電圧アラート|
|3|UVA|低電圧アラート|
|4|OIA|過電流アラート|

なお基板内非常停止は遠隔非常停止や過電圧・過電流非常停止など、ソフトウェアによる非常停止全般を指す。  
PCU_STATEはMONITORなどの設定が無い場合でも異常があった場合即座に発報する。  

### EX_EMS_TRG

各異常が発生した際に基板内非常停止を入れるかの選択

|bit|名称|概要|
|:--:|:--:|:--:|
|0:1|-|予約済み|
|2|OVA_EMS_EN|過電圧アラート|
|3|UVA_EMS_EN|低電圧アラート|
|4|OIA_EMS_EN|過電流アラート|

## ロボマス制御基板データ（Data type=0x2）  

nはモーターID（0~3）  
cは全モーター共通（0x105も0x205も結果は同じ）

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0xn00|NOP||||
|0xn01|MOTOR_TYPE|uint8_t|r/w|**未実装**|
|0xn02|CONTROL_TYPE|uint8_t|r/w|enum CONTROL_TYPE|
|0xn03|GEAR_RATIO|float|r/w|モーターのギア比|
|0xn04|MOTOR_STATE|uint8_t|r|**未実装**|
|0xc0F|CAN_TIMEOUT|uint16_t|r/w|この時間CAN信号が送られてこなければ停止 0で無効化|
|0xn10|PWM|float|r|現在のPWM|
|0xn11|PWM_TARGET|float|r/w|PWM指令値|
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
|0xcF0|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0xnF1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~0x3F|

現在位置は上書き可能。  
たとえばPOSに0を書きこめば現在位置が原点となる  

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

### CAN_TIMEOUT

ここで設定した時間CANが受信できなければ自動的に全モータはPWMモードに移行し停止する  
CAN復活時の自動再設定などは要望があれば実装するかも

### MONITOR_PERIOD/REG

MONITOR_PERIODで設定した周期でフィードバックするデータを選択する。  

なお周期は全モーター共通なので**個別に設定することはできない。**  
0x1F0に書きこもうと0x2F0に書きこもうと動作は同じになる。  

MONITOR_REGで設定したデータが定期的に送信される。  
たとえば現在の速度を定期的に送ってほしければMONITOR_REGの0x20ビット目を1にして書きこむ。  

## GPIO基板（Data type = 0x3）

nは操作したいピン番号

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x00|NOP||||
|0x01|PORT_READ|uint16_t|r|各PORTの現在の状態|
|0x03|PORT_WRITE|uint16_t|w|OUTPUTモードにしたPORTの出力 PICのLAT|
|0x1n|PWM_PERIOD|uint16_t|w|ソフトウェアPWMの周期|
|0x2n|PWM_DUTY|uint16_t|w|ソフトウェアPWMのduty|
|0xF0|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位) 0で停止|
|0xF1|MONITOR_REG|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~0x3F|

MODEは0で出力(PWM)、1で入力モード

各ピンは出力モードにした時、ソフトウェアPWMとして駆動することができる。  

### ソフトウェアPWMの動作

PWM出力に用いるカウンタは50kHzでカウントアップしている。  
たとえば50Hz、duty比30%（サーボ用PWM）を出力する場合、  

$$
\text{PWM\_PERIOD}=\frac{1}{50[Hz]}/\frac{1}{50000[Hz]}=1000
$$

$$
\text{PWM\_DUTY}=\text{PWM\_PERIOD}*0.3=300
$$

と設定すればよい。  
ピンに対応するPORT_WRITEレジスタのビットが1の時PWM出力され、0の時は出力されず、LOWとなる。  
PWMを出力せず単にGPIOとして動作させたい場合はDuty比100%となるようPWM_DUTYを設定（0xFFFFなど）すれば良い。  
