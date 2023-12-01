# 学ロボ2024 CAN/Serialプロトコル  

### CANの場合  

| bit | 名称 | 概要 |
|:--:|:--:|:--:|
|28:26|Priority level|優先度（0で最高）|
|25:22|Data type|データの種類|
|21:18|Board ID|基板の固有ID(0~F)|
|17:12|予約済み|0埋め推奨|
|11:8|Device ID|基板につながるデバイスを指定|
|7:0|Register ID||

なお、Board typeは次の様
| 名称 | ID |
|:--:|:--:|
| デバイスありますか | 0x0 |
| デバイスありますよ|0x1|
| 電源基板用データ | 0x2 |
| ロボマス制御用データ | 0x3 |
| 汎用GPIO用データ | 0x4 |

Board IDはロータリーDIPなどで指定される値。  

Device IDは基板に接続されているデバイスの番号（ロボマス制御基板につながるID3のロボマスモタドラに関して触りたいなら0x3にする、的な）。  
あとDevice ID=0は基板自体に関してｳﾝﾀﾗｶﾝﾀﾗしたいとき用。  

なお、Data type=0は接続デバイス確認用に定義。  
これを受信した基板はData type=1として利用可能なdevice IDを含めたIDで返答。  
つまりID1とID3のロボマスモタドラが接続されているBoard ID=3のロボマスモタドラ制御基板が0xP00を受信したら(0xP13 <<18)|(0x100) と(0xP13 <<18)|(0x300) を返答する。  

### Serialの場合  

|byte| bit | 名称 | 概要 |
|:--:|:--:|:--:|:--:|
|0|7|r/w|リモートフレーム枠|
|0|6:4|priority level||
|0|3:0|Data type|データの種類|
|1|7:4|Board ID|基板の固有ID|
|1|3:0|Device ID||
|2|7:0|Register ID||

とまあこんな感じで生成されるデータをやりとりするが、COBS形式にエンコードしてもいいかもしれないし、そもそもバイナリじゃなくて文字データの方が便利そう感はある。  

## ロボマス制御基板（金将）  

|Register ID |name|data type|r/w?|概要|
|:--:|:--:|:--:|:--:|:--:|
|0x0|NOP|||
|0x1|MOTOR_TYPE|uint8_t|r/w|enum MOTOR_TYPE|
|0x2|CONTROL_TYPE|uint8_t|r/w|enum CONTROL_TYPE|
|0x3|MOTOR_STATE|uint8_t|r/w|現在のモーターの状態表示|
|0x10|PWM|float(-1~1)|r|現在のPWM|
|0x11|PWM_LIM|float|r/w|トルクリミッタ的な|
|0x12|PWM_TARGET|float(-1~1)|w/r|
|0x20|SPD|float(rad/s)|r|現在の速度|
|0x21|SPD_LIM|float(rad/s)|r/w|速度制限|
|0x22|SPD_TARGET|float(rad/s)|r/w|目標速度|
|0x23|SPD_GAIN_P|float|r/w|速度Pゲイン|
|0x24|SPD_GAIN_I|float|r/w|速度Iゲイン|
|0x25|SPD_GAIN_D|float|r/w|速度Dゲイン|
|0x30|POS|flaot(rad/s)|r/w|現在の位置|
|0x31|POS_LIM|float(rad/s)|r/w|回転数制限|
|0x32|POS_TARGET|float(rad/s)|r/w|目標位置|
|0x33|POS_GAIN_P|float|r/w|位置Pゲイン|
|0x34|POS_GAIN_I|float|r/w|位置Iゲイン|
|0x35|POS_GAIN_D|float|r/w|位置Dゲイン|
|0x40|MONITOR_PERIOD|uint16_t|r/w|データをフィードバックする周期(1ms単位)|
|0x41|MONITOR_REG1|uint64_t|r/w|モニターするレジスタを設定 reg ID 0~63|
|0x42|MONITOR_REG2|uint64_t|r/w|モニターするレジスタを設定 reg ID 64~123|

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
