# 学ロボ2024 CAN/Serialプロトコル  

### CANの場合  

| bit | 名称 | 概要 |
|:--:|:--:|:--:|
|28:26|Priority level|優先度（0で最高）|
|25:22|Data type|データの種類|
|21:18|Board ID|基板の固有ID(0~F)|
|17:16|予約済み|0埋め推奨|
|15:0|Register ID||

なお、Data typeは次の様
| 名称 | ID |
|:--:|:--:|
| 共通データ形式（強制受信） | 0xFF |
| 共通データ形式 | 0x0 |
| 電源基板用データ | 0x1 |
| ロボマス制御用データ | 0x2 |
| 汎用GPIO用データ | 0x3 |

Board IDはロータリーDIPなどで指定される値。  
ID=0x0の場合はboard_idの一致不一致関わらず受信し処理する。

### Serialの場合  

|byte| bit | 名称 | 概要 |
|:--:|:--:|:--:|:--:|
|0|3|r/w|リモートフレーム枠|
|0|2:0|priority level||
|1|7:4|Data type|データの種類|
|1|3:0|Board ID|基板の固有ID|
|2|7:0|Register ID 15~8||
|3|7:0|Register ID 7~0||

0byte目はパケットの情報、1~3byte目はデータの情報的な

実際にはCOBS形式にエンコードしてやり取りする

## 共通データ形式（Data type=0x0,0x1）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0|REQEST_ID|-|-|これを受信したデバイスは自身のIDを返せ|
|0x1|RESPONSE_ID|uint8_t|-|データ部分は基板形式|
|0xE|EMERGENCY_STOP|uint8_t|-|非常停止|
|0xF|RESET_EMERGENCY_STOP|uint8_t|-|非常停止解除|

REQEST_IDに対して返答する際は、Board IDに自身のID（ロータリーDIPの値）、データに基板の形式を入れてData type 0x0で返す。
基板形式は以下の通り。  
| 名称 | ID |
|:--:|:--:|
| 電源基板 | 0x1 |
| ロボマス制御基板 | 0x2 |
| 汎用GPIO基板 | 0x3 |

## ロボマス制御基板データ（Data type=0x1）  

nはモーターID（1~4）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0xn00|NOP|||
|0xn01|MOTOR_TYPE|uint8_t|r/w|enum MOTOR_TYPE|
|0xn02|CONTROL_TYPE|uint8_t|r/w|enum CONTROL_TYPE|
|0xn03|MOTOR_STATE|uint8_t|r/w|現在のモーターの状態表示|
|0xn10|PWM|float(-1~1)|r|現在のPWM|
|0xn11|PWM_LIM|float|r/w|トルクリミッタ的な|
|0xn12|PWM_TARGET|float(-1~1)|w/r|
|0xn20|SPD|float(rad/s)|r|現在の速度|
|0xn21|SPD_LIM|float(rad/s)|r/w|速度制限|
|0xn22|SPD_TARGET|float(rad/s)|r/w|目標速度|
|0xn23|SPD_GAIN_P|float|r/w|速度Pゲイン|
|0xn24|SPD_GAIN_I|float|r/w|速度Iゲイン|
|0xn25|SPD_GAIN_D|float|r/w|速度Dゲイン|
|0xn30|POS|flaot(rad/s)|r/w|現在の位置|
|0xn31|POS_LIM|float(rad/s)|r/w|回転数制限|
|0xn32|POS_TARGET|float(rad/s)|r/w|目標位置|
|0xn33|POS_GAIN_P|float|r/w|位置Pゲイン|
|0xn34|POS_GAIN_I|float|r/w|位置Iゲイン|
|0xn35|POS_GAIN_D|float|r/w|位置Dゲイン|
|0xn40|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位)|
|0xn41|MONITOR_REG1|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~63|
|0xn42|MONITOR_REG2|uint64_t|r/w|モニターするレジスタを設定 reg ID 64~123|

現在位置は上書き可能。  

### MOTOR_TYPE

||名称|備考|
|:--:|:--:|:--:|
|0x0|M2006||
|0x1|M3508||
|0x2|VESC||

### CONTROL_MODE

||名称|備考|
|:--:|:--:|:--:|
|0x0|PWM_MODE|オープンループ|
|0x1|SPEED_MODE|速度制御|
|0x2|POSITION_MODE|位置制御|
|0x3|ABS_POSITION_MODE|絶対位置制御（AS5600利用）|

### MOTOR_STATE

|bit|名称|備考|
|:--:|:--:|:--:|
|7|MD_IS_ACTIVE|ドライバが有効であれば1|
|6|STALL|モーターがストールしたら1|
|5|TH_SD|過熱保護状態で1|
|4|OCD|過電流状態で1|
|1:0|ROTATE|停止で0正回転で1逆回転で2|


ストール検知について
速度制御・位置制御モードにおいてPWMがPWM_LIMに引っかかっている、かつ速度が0またはPWMの指令と逆の時。  
PWM_MODEの時は無効。  
とおもったけどそもそも自動整合型PIDだと引っかかるとかいう概念が存在しないのでもう少しロジックは詰めないといけない。  

### MONITOR_REG

MONITOR_PERIODで設定した周期でフィードバックするデータを選択する。  

たとえば現在の速度を送ってほしければMONITOR_REGの0x20ビット目を1にして書きこむ。  
