# M5StickS3とMicroPythonで振動測定とFFT解析（UIFlow2＋Thonny）

## はじめに

機械の振動を調べる際には、振動の大きさだけでなく、どのような周波数の振動が発生しているかを知ることも重要です。

以前、M5StickS3に内蔵されている加速度センサを使い、Arduinoで振動測定とFFT解析を行いました。

今回は、同じM5StickS3を使い、MicroPythonでも振動測定ができるか試してみました。

MicroPythonはPythonに似た文法でプログラムを作成できる一方、一般にArduinoで作成したネイティブコードより処理速度が遅くなる場合があります。

そこで今回は、加速度センサのFIFO機能を利用して振動データを取得し、FFT（高速フーリエ変換）によって周波数分析を行いました。

## 1. 使用した機器と開発環境

| 項目 | 内容 |
|---|---|
| マイコン | M5Stack M5StickS3 |
| 加速度センサ | BMI270（本体内蔵） |
| プログラム言語 | MicroPython |
| 開発環境 | Thonny |
| ファームウェア | UIFlow2 |
| 測定方向 | Z軸 |

M5StickS3にはBMI270加速度センサが内蔵されているため、外付けのセンサは必要ありません。

MicroPythonの環境構築については、別記事「M5StickS3でMicroPythonを使うための環境構築（UIFlow2＋Thonny）」を参照してください。

## 2. 作成した振動測定プログラム

今回のプログラムには、次の機能を持たせました。

- Aボタンによる測定開始・停止
- BMI270によるZ軸方向の加速度測定
- FIFO機能を利用した1024点のデータ取得
- FFTによる周波数分析
- 5～200 Hzの範囲から振幅の大きい3つのピーク周波数を検出
- 振動加速度の実効値（RMS）の計算
- 測定結果のディスプレイ表示
- CSV形式での測定結果の保存

プログラムを `main.py` として保存し、起動設定を変更することで、パソコンを接続せずに測定できるようにしました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/47c39cb5-ef24-49e7-a1d3-63c1a290e630.png)

【写真：振動測定中のM5StickS3】

## 3. FIFO機能を使った振動データの取得

今回のポイントは、加速度センサのFIFO機能を使用したことです。

MicroPythonで加速度センサの値を1点ずつ読み出す場合、Python側の処理時間などによって、データを一定間隔で取得することが難しくなる場合があります。

一方、BMI270には、センサ内部で測定したデータを一時的に蓄積するFIFO機能があります。

今回はこの機能を利用し、センサ側で一定間隔に取得されたデータをまとめて読み出しました。

これにより、MicroPythonによる読み出し処理の影響を抑えながら、連続した振動データを取得できます。

今回の主な測定条件は次のとおりです。

| 項目 | 設定・測定値 |
|---|---:|
| 加速度センサ | BMI270 |
| データ取得方式 | FIFO |
| サンプリング周波数 | 約809.6 Hz |
| FFTデータ数 | 1024点 |
| 周波数分解能 | 約0.791 Hz |
| ピーク検出範囲 | 5～200 Hz |

周波数分解能は、サンプリング周波数をデータ数で割った値です。

$$
\Delta f=\frac{809.6}{1024}\approx0.791\ \mathrm{Hz}
$$

この条件では、約0.791 Hz間隔でFFTの周波数成分を求めることができます。

## 4. 振動加速度の実効値とFFT解析

取得した加速度データから平均値を差し引き、振動成分の実効値（RMS）を求めました。

$$
{\mathrm{RMS}}=
\sqrt{\frac{1}{N}\sum_{i=1}^{N}(a_i-\bar a)^2}
$$

ここで、\(a_i\) は各測定点の加速度、\(\bar a\) は加速度の平均値、\(N\) はデータ数です。

平均値を差し引くことで、重力加速度などの直流成分を除いています。

また、取得した1024点の加速度データについてFFT解析を行い、5～200 Hzの範囲で振幅の大きい3つのピーク周波数を検出しました。

これにより、振動の大きさだけでなく、主な周波数成分も確認できます。

## 5. MicroPythonプログラム

プログラムはThonnyで作成し、M5StickS3に `main.py` として保存しました。

プログラムでは、BMI270の初期設定、FIFOからのデータ取得、FFT計算、ピーク検出、画面表示、CSV保存を行っています。

<details><summary>プログラムはこちらを参照ください</summary>

```python

import M5
from M5 import *
from machine import Pin, SoftI2C
import time
import math
import os

# ==========================================
# M5StickS3 + BMI270
# MicroPython FFT Vibration Logger
#
# 1024-point FFT
# Automatic top 3 peak detection
# Button A: START / STOP
# CSV logging
# ==========================================

N = 1024
FS = 809.6

FREQ_MIN = 5.0
FREQ_MAX = 200.0
PEAK_COUNT = 3
MIN_PEAK_DISTANCE = 3.0

WAIT_MS = 300

BMI_ADDR = 0x68
ACC_SCALE = 4096.0  # +/-8g

# ------------------------------------------
# M5StickS3 initialization
# ------------------------------------------
M5.begin()
M5.Display.setRotation(1)

button_a = Pin(11, Pin.IN, Pin.PULL_UP)

# BMI270 internal I2C
i2c = SoftI2C(
    sda=Pin(47),
    scl=Pin(48),
    freq=400000
)


# ------------------------------------------
# BMI270 register access
# ------------------------------------------
def read_reg(reg):
    return i2c.readfrom_mem(
        BMI_ADDR, reg, 1
    )[0]


def write_reg(reg, value):
    i2c.writeto_mem(
        BMI_ADDR, reg, bytes([value])
    )


def init_bmi270():
    chip_id = read_reg(0x00)
    print("CHIP_ID:", hex(chip_id))

    if chip_id != 0x24:
        raise RuntimeError("BMI270 not found")

    # UIFlow2 firmware initializes BMI270.
    # Configure accelerometer and FIFO.
    write_reg(0x40, 0xAB)  # ODR 800 Hz
    write_reg(0x41, 0x02)  # +/-8g
    write_reg(0x48, 0x00)  # Headerless FIFO
    write_reg(0x49, 0x40)  # ACC FIFO
    write_reg(0x7E, 0xB0)  # Flush FIFO

    time.sleep_ms(50)


def fifo_length():
    low = read_reg(0x24)
    high = read_reg(0x25) & 0x3F
    return (high << 8) | low


def signed16(low, high):
    value = low | (high << 8)
    if value >= 32768:
        value -= 65536
    return value


# ------------------------------------------
# Acquire 1024 Z-axis samples
# ------------------------------------------
def acquire_samples():
    write_reg(0x7E, 0xB0)
    time.sleep_ms(5)

    samples = []
    start = time.ticks_ms()

    while len(samples) < N:

        available = fifo_length() // 6

        if available >= 4:
            frames = min(
                4, available, N - len(samples)
            )

            data = i2c.readfrom_mem(
                BMI_ADDR, 0x26, frames * 6
            )

            for j in range(frames):
                p = j * 6
                z = signed16(
                    data[p + 4],
                    data[p + 5]
                )
                samples.append(z / ACC_SCALE)

        else:
            time.sleep_ms(1)

        if time.ticks_diff(
            time.ticks_ms(), start
        ) > 5000:
            raise RuntimeError("FIFO timeout")

    elapsed = time.ticks_diff(
        time.ticks_ms(), start
    )

    return samples, elapsed


# ------------------------------------------
# FFT: radix-2, in-place
# ------------------------------------------
def fft(real, imag):
    n = len(real)

    # Bit reversal
    j = 0
    for i in range(1, n):
        bit = n >> 1

        while j & bit:
            j ^= bit
            bit >>= 1

        j ^= bit

        if i < j:
            real[i], real[j] = real[j], real[i]
            imag[i], imag[j] = imag[j], imag[i]

    # Butterfly
    size = 2

    while size <= n:
        half = size // 2
        angle = -2.0 * math.pi / size

        wm_r = math.cos(angle)
        wm_i = math.sin(angle)

        for start in range(0, n, size):
            wr = 1.0
            wi = 0.0

            for k in range(half):
                a = start + k
                b = a + half

                tr = wr * real[b] - wi * imag[b]
                ti = wr * imag[b] + wi * real[b]

                ur = real[a]
                ui = imag[a]

                real[a] = ur + tr
                imag[a] = ui + ti

                real[b] = ur - tr
                imag[b] = ui - ti

                new_wr = wr * wm_r - wi * wm_i
                wi = wr * wm_i + wi * wm_r
                wr = new_wr

        size *= 2


# ------------------------------------------
# Automatic top 3 peak detection
# ------------------------------------------
def find_top_peaks(amplitudes):
    df = FS / N

    k_min = max(
        1, int(math.ceil(FREQ_MIN / df))
    )
    k_max = min(
        len(amplitudes) - 2,
        int(FREQ_MAX / df)
    )

    candidates = []

    # Local maxima only
    for k in range(k_min, k_max + 1):
        a = amplitudes[k]

        if (
            a > amplitudes[k - 1]
            and a >= amplitudes[k + 1]
        ):
            candidates.append((k * df, a))

    # Largest amplitude first
    candidates.sort(
        key=lambda p: p[1],
        reverse=True
    )

    peaks = []

    for freq, amp in candidates:
        too_close = False

        for selected_freq, selected_amp in peaks:
            if abs(freq - selected_freq) < MIN_PEAK_DISTANCE:
                too_close = True
                break

        if not too_close:
            peaks.append((freq, amp))

        if len(peaks) >= PEAK_COUNT:
            break

    return peaks


# ------------------------------------------
# FFT analysis
# ------------------------------------------
def analyze(samples):
    t0 = time.ticks_ms()

    mean = sum(samples) / N

    ac = [x - mean for x in samples]

    rms = math.sqrt(
        sum(x * x for x in ac) / N
    )

    real = [0.0] * N
    imag = [0.0] * N

    # Hann window
    window_sum = 0.0

    for i in range(N):
        w = 0.5 * (
            1.0 - math.cos(
                2.0 * math.pi * i / (N - 1)
            )
        )
        real[i] = ac[i] * w
        window_sum += w

    fft(real, imag)

    amplitudes = [0.0] * (N // 2 + 1)

    for k in range(1, N // 2):
        amplitudes[k] = (
            2.0
            * math.sqrt(
                real[k] * real[k]
                + imag[k] * imag[k]
            )
            / window_sum
        )

    peaks = find_top_peaks(amplitudes)

    fft_ms = time.ticks_diff(
        time.ticks_ms(), t0
    )

    return rms, peaks, fft_ms


# ------------------------------------------
# Display
# ------------------------------------------


def show_result(status, count, rms, peaks):
    d = M5.Display

    d.fillScreen(0x000000)
    d.setTextColor(0xFFFFFF)

    # タイトル（小さい文字）
    d.setTextSize(1)
    d.drawString(
        "FFT {} #{}".format(status, count),
        5, 5
    )

    # RMSとピーク周波数（大きい文字）
    d.setTextSize(2)

    d.drawString(
        "RMS {:.4f}g".format(rms),
        5, 25
    )

    for i in range(PEAK_COUNT):
        y = 53 + i * 27

        if i < len(peaks):
            freq, amp = peaks[i]
            s = "P{}: {:.2f}Hz".format(
                i + 1, freq
            )
        else:
            s = "P{}: ---".format(i + 1)

        d.drawString(s, 5, y)

    # 操作案内（小さい文字）
    d.setTextSize(1)
    d.drawString(
        "A:STOP" if status == "RUN"
        else "A:START",
        5, 145
    )


def show_ready():
    M5.Display.fillScreen(0x000000)
    M5.Display.setTextColor(0xFFFFFF)
    M5.Display.setTextSize(2)
    M5.Display.drawString("FFT READY", 5, 10)
    M5.Display.drawString("5-200 Hz", 5, 50)
    M5.Display.drawString("Press A", 5, 90)


def show_error(message):
    M5.Display.fillScreen(0x000000)
    M5.Display.setTextColor(0xFFFFFF)
    M5.Display.setTextSize(2)
    M5.Display.drawString("ERROR", 5, 10)
    M5.Display.setTextSize(1)
    M5.Display.drawString(
        str(message)[:30], 5, 50
    )


# ------------------------------------------
# CSV file
# ------------------------------------------
def new_filename():
    for number in range(1, 1000):
        filename = "vibration_{:03d}.csv".format(
            number
        )

        try:
            os.stat(filename)
        except OSError:
            return filename

    raise RuntimeError("File limit reached")


def save_result(f, count, elapsed_s,
                rms, peaks, acquire_ms, fft_ms):

    fields = [
        str(count),
        "{:.2f}".format(elapsed_s),
        "{:.6f}".format(rms)
    ]

    for i in range(PEAK_COUNT):
        if i < len(peaks):
            freq, amp = peaks[i]
            fields.append("{:.3f}".format(freq))
            fields.append("{:.6f}".format(amp))
        else:
            fields.extend(["", ""])

    fields.append(str(acquire_ms))
    fields.append(str(fft_ms))

    f.write(",".join(fields) + "\n")
    f.flush()


# ------------------------------------------
# Button detection
# ------------------------------------------
last_button = button_a.value()
last_press_ms = time.ticks_ms()


def button_pressed():
    global last_button, last_press_ms

    current = button_a.value()
    now = time.ticks_ms()

    pressed = (
        last_button == 1
        and current == 0
        and time.ticks_diff(
            now, last_press_ms
        ) > 200
    )

    last_button = current

    if pressed:
        last_press_ms = now

    return pressed


# ------------------------------------------
# Main
# ------------------------------------------
running = False
csv_file = None
csv_filename = ""

count = 0
rms = 0.0
peaks = []

start_ms = 0
next_ms = 0

try:
    init_bmi270()
    show_ready()

    print("FFT Vibration Logger")
    print("Frequency range: 5-200 Hz")
    print("Press A to START / STOP")

    while True:

        if button_pressed():

            if not running:
                csv_filename = new_filename()
                csv_file = open(csv_filename, "w")

                csv_file.write(
                    "No,Time_s,RMS_g,"
                    "Peak1_Hz,Amp1_g,"
                    "Peak2_Hz,Amp2_g,"
                    "Peak3_Hz,Amp3_g,"
                    "Acquire_ms,FFT_ms\n"
                )
                csv_file.flush()

                count = 0
                rms = 0.0
                peaks = []

                start_ms = time.ticks_ms()
                next_ms = start_ms

                running = True

                show_result(
                    "RUN", count, rms, peaks
                )

                print("START:", csv_filename)

            else:
                running = False

                if csv_file is not None:
                    csv_file.close()
                    csv_file = None

                show_result(
                    "STOP", count, rms, peaks
                )

                print("STOP:", csv_filename)

        if running:

            now = time.ticks_ms()

            if time.ticks_diff(
                now, next_ms
            ) >= 0:

                samples, acquire_ms = acquire_samples()

                rms, peaks, fft_ms = analyze(
                    samples
                )

                count += 1

                elapsed_s = time.ticks_diff(
                    time.ticks_ms(), start_ms
                ) / 1000

                save_result(
                    csv_file,
                    count,
                    elapsed_s,
                    rms,
                    peaks,
                    acquire_ms,
                    fft_ms
                )

                show_result(
                    "RUN", count, rms, peaks
                )

                print(
                    "No={}, RMS={:.5f}g".format(
                        count, rms
                    )
                )

                for i, (freq, amp) in enumerate(peaks):
                    print(
                        " P{}: {:.2f}Hz, {:.5f}g".format(
                            i + 1, freq, amp
                        )
                    )

                next_ms = time.ticks_add(
                    time.ticks_ms(), WAIT_MS
                )

        time.sleep_ms(30)

except Exception as e:
    print("ERROR:", e)
    show_error(e)

finally:
    if csv_file is not None:
        csv_file.close()
        print("CSV saved:", csv_filename)
```
</details>

プログラムを保存した後、M5BurnerのBoot Optionを `Run main.py directly` に設定すると、M5StickS3単体で動作させることができます。

## 6. 振動の測定結果

動作確認のため、スマートフォンから110 Hz、150 Hz、195 Hzの音を同時に出力しました。

スマートフォンの背面を上にして置き、その上にM5StickS3を載せ、Z軸方向の振動を測定しました。

同じ条件で5回測定した結果を示します。

| 音波の周波数 | 検出周波数（5回平均） |
|---|---:|
| 110 Hz | 109.90 Hz |
| 150 Hz | 150.22 Hz |
| 195 Hz | 195.28 Hz |

3つの周波数成分をそれぞれ検出することができました。

また、5回の測定では、各成分とも同じ周波数が検出されました。

ただし、今回のFFTの周波数分解能は約0.791 Hzです。表中の小数第2位までの表示が、そのまま周波数の測定精度を示しているわけではありません。

今回の結果から、MicroPythonを使った場合でも、複数の振動周波数を検出できることが確認できました。

## 7. MicroPythonによるFFTの処理時間

今回の条件では、1024点のFFT計算に約1.5秒かかりました。

このため、FFT解析を高速に繰り返す用途には制約があります。

一方、今回のように一定時間の振動データを取得し、その周波数成分を確認する用途であれば、MicroPythonでも利用できる可能性があります。

今回の実験では、FIFO機能を使って加速度データを取得したことが、MicroPythonによる振動測定を実現するうえで重要だったと考えています。

## 8. 使ってみた感想

今回、M5StickS3の内蔵加速度センサを使い、MicroPythonでも振動測定とFFT解析ができることを確認しました。

Arduinoと比較すると、MicroPythonは処理速度の面で不利になる場合があります。しかし、センサ側のFIFO機能を利用することで、一定間隔で取得された振動データをまとめて読み出せます。

また、MicroPythonはプログラムを修正してすぐに実行できるため、測定条件を変更しながら試行錯誤する用途にも向いていると感じました。

今回の測定器は、校正された振動計ではありません。振動加速度の測定精度や測定可能な周波数範囲については、用途に応じた確認が必要です。

それでも、身近なマイコンで振動の周波数成分を確認できることは、機械の振動測定に取り組む最初の一歩として面白いと思います。

## おわりに

今回はM5StickS3に内蔵されたBMI270加速度センサを使用し、MicroPythonによる振動測定とFFT解析を行いました。

FIFO機能を利用して1024点の加速度データを取得し、FFTによって110 Hz、150 Hz、195 Hzの3つの周波数成分を検出することができました。

MicroPythonでも、このような簡単な振動測定器を作れることが分かりました。

Pythonに慣れている人にとっては、マイコンを使った振動測定に取り組む選択肢の一つになると思います。

## 参考資料

- [M5Stack公式ドキュメント](https://docs.m5stack.com/)
- [MicroPython公式サイト](https://micropython.org/)
- [Thonny公式サイト](https://thonny.org/)
