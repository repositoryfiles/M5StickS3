# M5StickS3とMicroPythonで温湿度データロガーを作る（UIFlow2＋Thonny）

## はじめに

製造現場では、温度や湿度を記録することで、作業環境の把握や製品品質の管理に役立てることができます。

今回は、小型マイコンのM5StickS3と温湿度センサを使い、MicroPythonで簡単な温湿度データロガーを作ってみました。

M5StickS3にはディスプレイとバッテリーが内蔵されているため、パソコンを接続せずに測定できます。

今回作成したプログラムでは、Aボタンで測定を開始・停止し、60秒間隔で温湿度を測定してCSVファイルに保存します。

## 1. 使用した機器

| 機器 | 内容 |
|---|---|
| マイコン | M5Stack M5StickS3 |
| 温湿度センサ | M5Stack ENV III Unit |
| 接続 | Groveケーブル |
| 開発環境 | Thonny |
| プログラム言語 | MicroPython |
| ファームウェア | UIFlow2 |

ENV IIIにはSHT30温湿度センサが搭載されています。

M5StickS3とENV IIIをGroveケーブルで接続するだけなので、はんだ付けや複雑な配線は必要ありません。

![PXL_20261031_090352152.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/d2101f40-d7a4-40b3-b8e8-c7e8d09f952a.jpeg)
【写真：M5StickS3とENV IIIを接続した状態】

MicroPythonの環境構築については、別記事「M5StickS3でMicroPythonを使うための環境構築（UIFlow2＋Thonny）」を参照してください。

## 2. 作成したプログラムの機能

今回のプログラムには、次の機能を持たせました。

- Aボタンを押すと測定を開始し、再度押すと停止
- 60秒間隔で温度と相対湿度を測定
- 測定値と測定回数をディスプレイに表示
- 測定データをCSV形式で本体に保存
- 測定停止後も最後の測定値を表示
- 測定を開始するたびに新しいCSVファイルを作成

測定を停止するとファイルを閉じるため、保存したデータをパソコンで読み出せます。

## 3. 温湿度の測定方法

ENV IIIのSHT30は、I2C通信で温湿度データを取得できます。

今回のプログラムでは、M5StickS3のGrove端子に接続したセンサからデータを読み出します。

主な設定は次のとおりです。

| 項目 | 設定 |
|---|---|
| I2C SDA | GPIO9 |
| I2C SCL | GPIO10 |
| SHT30アドレス | 0x44 |
| 測定間隔 | 60秒 |

SHT30では、センサから取得したデータを次の式で温度と相対湿度に換算します。

温度：

$$
T=-45+175\frac{S_T}{65535}
$$

相対湿度：

$$
RH=100\frac{S_{RH}}{65535}
$$

ここで、\(S_T\)、\(S_{RH}\) はそれぞれセンサから取得した16ビットの測定値です。

## 4. プログラムの作成と保存

ThonnyでMicroPythonプログラムを作成し、M5StickS3に `main.py` として保存します。

プログラムでは、温湿度測定、画面表示、Aボタンによる開始・停止、CSVファイルへの保存を行います。

<details><summary>プログラムはこちらを参照ください</summary>

```python
import M5
from M5 import *
from machine import I2C, Pin
import time

# =====================================
# M5StickS3 + ENV III 温湿度ロガー
# UIFlow2 / MicroPython
#
# Aボタンで測定開始・停止
# 60秒間隔でCSV保存
# =====================================

INTERVAL_S = 60

# M5StickS3初期化
M5.begin()
M5.Display.setRotation(1)
M5.Display.setTextSize(2)

# ENV III（現在の動作確認済み設定を維持）
i2c = I2C(
    0,
    sda=Pin(9),
    scl=Pin(10),
    freq=100000
)
SHT30_ADDR = 0x44

# Aボタン（GPIO11）
button_a = Pin(11, Pin.IN, Pin.PULL_UP)


# -------------------------------------
# 温湿度測定
# -------------------------------------
def read_sht30():
    i2c.writeto(SHT30_ADDR, b'\x2c\x06')
    time.sleep_ms(20)

    data = i2c.readfrom(SHT30_ADDR, 6)

    raw_t = (data[0] << 8) | data[1]
    raw_h = (data[3] << 8) | data[4]

    temperature = -45 + 175 * raw_t / 65535
    humidity = 100 * raw_h / 65535

    return temperature, humidity


# -------------------------------------
# 画面表示
# -------------------------------------
def show_values(temp, hum, count, status):
    M5.Display.fillScreen(0x000000)
    M5.Display.setTextColor(0xFFFFFF)

    M5.Display.drawString(status, 5, 5)

    if temp is not None:
        M5.Display.drawString(
            "Temp: {:.2f} C".format(temp), 5, 40
        )
        M5.Display.drawString(
            "Hum : {:.2f} %".format(hum), 5, 75
        )

    M5.Display.drawString(
        "Count: {}".format(count), 5, 110
    )


# -------------------------------------
# CSVファイル名を自動生成
# -------------------------------------
def new_csv_filename():
    import os

    for number in range(1, 1000):
        filename = "temp_{:03d}.csv".format(number)

        try:
            os.stat(filename)
        except OSError:
            return filename

    raise OSError("CSV file limit reached")


# -------------------------------------
# Aボタンの押下検出
# チャタリング対策
# -------------------------------------
last_button = button_a.value()
last_press_ms = time.ticks_ms()

def button_pressed():
    global last_button, last_press_ms

    current = button_a.value()
    now = time.ticks_ms()

    pressed = (
        last_button == 1
        and current == 0
        and time.ticks_diff(now, last_press_ms) > 200
    )

    last_button = current

    if pressed:
        last_press_ms = now

    return pressed


# -------------------------------------
# メインプログラム
# -------------------------------------
running = False
csv_file = None
csv_filename = ""

count = 0
temp = None
hum = None

start_ms = 0
next_ms = 0

show_values(temp, hum, count, "READY")

print("Temperature/Humidity Logger")
print("Press A to START / STOP")

try:
    while True:

        # Aボタンで開始・停止
        if button_pressed():

            if not running:
                # 測定開始
                csv_filename = new_csv_filename()
                csv_file = open(csv_filename, "w")

                csv_file.write(
                    "Time_s,Temperature_C,Humidity_pct\n"
                )
                csv_file.flush()

                count = 0
                temp = None
                hum = None

                start_ms = time.ticks_ms()
                next_ms = start_ms

                running = True

                show_values(
                    temp, hum, count, "LOGGING"
                )

                print("Measurement started.")
                print("CSV:", csv_filename)

            else:
                # 測定停止
                running = False

                if csv_file is not None:
                    csv_file.close()
                    csv_file = None

                show_values(
                    temp, hum, count, "STOPPED"
                )

                print("Measurement stopped.")
                print("CSV saved:", csv_filename)

        # 測定中のみ温湿度を取得
        if running:

            now = time.ticks_ms()

            if time.ticks_diff(now, next_ms) >= 0:

                temp, hum = read_sht30()

                elapsed_s = time.ticks_diff(
                    now, start_ms
                ) / 1000

                csv_file.write(
                    "{:.1f},{:.2f},{:.2f}\n".format(
                        elapsed_s, temp, hum
                    )
                )
                csv_file.flush()

                count += 1

                show_values(
                    temp, hum, count, "LOGGING"
                )

                print(
                    "{:.1f},{:.2f},{:.2f}".format(
                        elapsed_s, temp, hum
                    )
                )

                next_ms = time.ticks_add(
                    next_ms, INTERVAL_S * 1000
                )

        time.sleep_ms(50)

finally:
    # Thonnyから停止した場合もファイルを閉じる
    if csv_file is not None:
        csv_file.close()

    print("Program terminated.")

```
</details>

なお、UIFlow2の起動設定は、M5Burnerで `Run main.py directly` に変更しておきます。

これにより、M5StickS3の電源を入れるとプログラムが自動実行されます。

## 5. 測定結果

製作した温湿度データロガーの動作状況を示します。

![PXL_20261031_090420129.MACRO_FOCUS.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/b60d6f76-c063-4b68-b4ad-d4966341874d.jpeg)
【写真：温湿度と測定回数を表示したM5StickS3】

今回の測定では、ディスプレイに次の値が表示されました。

| 項目 | 表示値 |
|---|---:|
| 温度 | 22.4 ℃ |
| 相対湿度 | 61.9 % |
| 測定回数 | 1回 |

測定データはM5StickS3内のCSVファイルに保存され、測定終了後にThonnyを使ってパソコンへ読み出せることを確認しました。

CSVファイルはExcelなどでも開くことができるため、時間による温湿度の変化をグラフ化することもできます。

## 6. 使ってみた感想

今回、MicroPythonを使って温湿度の測定とデータ保存を行うことができました。

M5StickS3はディスプレイとバッテリーを内蔵しているため、別途表示器や電源回路を用意する必要がありません。

また、ENV IIIはGroveケーブルで接続できるため、電子回路に詳しくなくても比較的簡単に測定器を作ることができます。

今回のような仕組みは、製造現場での温湿度変化の確認や、簡単な環境データの記録などに活用できそうです。

ただし、品質管理などに利用する場合は、センサの精度や設置条件、バッテリーでの連続動作時間などを確認する必要があります。

## おわりに

今回はM5StickS3とENV IIIを使い、MicroPythonで温湿度データロガーを製作しました。

Aボタンによる測定開始・停止、ディスプレイへの表示、CSVファイルへの保存まで、M5StickS3単体で動作させることができました。

MicroPythonを使えば、Pythonに慣れた人でも比較的取り組みやすく、ちょっとした測定器を作るための選択肢になると思います。

今後も、M5StickS3とMicroPythonを使った簡単な測定を試してみたいと思います。

## 参考資料

- [M5Stack公式ドキュメント](https://docs.m5stack.com/)
- [MicroPython公式サイト](https://micropython.org/)
- [Thonny公式サイト](https://thonny.org/)

