# M5StickS3でMicroPythonを使うための環境構築（UIFlow2＋Thonny）

## はじめに

M5StickS3は、ディスプレイやバッテリーを内蔵した小型のマイコンです。センサを接続することで、温湿度や振動などの測定にも利用できます。

M5StickS3のプログラム開発にはArduino IDEなどが利用できますが、ここではPythonに似た文法でプログラムを作成できるMicroPythonを使用します。

開発環境にはThonnyを使用します。ThonnyはPython用の開発環境で、MicroPythonにも対応しています。

この記事では、**M5StickS3を購入した状態から、MicroPythonでプログラムを作成し、M5StickS3単体で実行するまで**の手順を紹介します。

## 1. 使用する機器とソフトウェア

今回使用した環境は以下のとおりです。

| 項目 | 内容 |
|---|---|
| マイコン | M5StickS3 |
| パソコン | Windows 11 |
| ファームウェア | UIFlow2 V2.5.5 |
| ファームウェア書き込みソフト | M5Burner |
| プログラム開発環境 | Thonny |
| 接続ケーブル | USB Type-C（データ通信対応） |

今回はWi-Fiによるプログラム開発は行わず、M5StickS3とパソコンをUSBケーブルで接続して使用します。

## 2. UIFlow2の書き込み

M5StickS3でMicroPythonを使用するため、UIFlow2のファームウェアを書き込みます。

なお、購入してすぐにmicropythonを使う場合は
3. Thonnyのインストール
に進んで下さい。

### 2.1 M5Burnerのインストール

M5Stackの公式サイトからM5Burnerをダウンロードし、パソコンにインストールします。

参考：[M5Burnerダウンロードページ](https://docs.m5stack.com/en/uiflow/m5burner/intro)

### 2.2 M5StickS3をダウンロードモードにする

1. M5StickS3をUSBケーブルでパソコンに接続します。
2. M5StickS3側面のリセットボタンを長押しします。
3. 内部の緑色LEDが点滅したら、ダウンロードモードになっています。

この操作は、後述する起動設定の変更時にも必要です。

### 2.3 ファームウェアを書き込む

1. M5Burnerを起動します。
2. M5StickS3用のUIFlow2を選択します。
3. ファームウェアをダウンロードします。
4. `Burn` をクリックします。
5. M5StickS3が接続されているCOMポートを選択します。
6. 書き込みを実行します。

今回はUIFlow2 V2.5.5を使用しました。

書き込みが完了したら、M5StickS3を再起動します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/43362847-e616-4f47-a942-0663a2841afe.png)
**Downloadをクリック**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/4cc427b5-c78d-481f-bd3c-9db043354500.png)
**Burnをクリック**

## 3. Thonnyのインストール

[Thonny公式サイト](https://thonny.org/)からWindows版をダウンロードし、インストールします。

Thonnyは、プログラムの作成・実行や、M5StickS3内のファイルの管理に使用します。

## 4. Thonnyの設定

M5StickS3をUSBケーブルでパソコンに接続し、Thonnyを起動します。

「ツール」→「オプション」→「インタプリタ」を開き、以下の設定を行います。

| 項目 | 設定 |
|---|---|
| インタプリタ | MicroPython (ESP32) |
| ポート | M5StickS3のCOMポート |
| スクリプトを実行する前にインタプリタを再起動する | チェックを外す |

COMポート番号はパソコンの環境によって異なります。今回の環境ではCOM6でした。
※COMポート番号はデバイスマネージャーのポート（COMとLPT）で確認できます。

**重要なのは、「スクリプトを実行する前にインタプリタを再起動する」のチェックを外すことです。**

今回、このチェックが入った状態でプログラムを実行すると、以下のメッセージが表示されました。

```text
MPY: soft reboot

uiflow V2.5.5
Startup with network type: WIFI
WiFi initialized
```

この状態ではUIFlow2の起動処理が実行され、Thonnyのシェルにプロンプトが表示されませんでした。

Ctrl+Cで処理を中断するとプログラムを実行できましたが、毎回この操作を行うのは不便です。

そこで、上記（スクリプトを実行する前にインタプリタを再起動する）のチェックを外したところ、F5キーで正常にプログラムを実行できるようになりました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/619944cf-80e4-4a29-8653-a23781a81581.png)
**Thonnyのインタプリタ設定画面**

なお、この設定ではプログラム実行前にインタプリタが再起動されないため、以前のプログラムで設定した変数などが残る場合があります。

## 5. MicroPythonプログラムの実行

Thonnyのエディタに、以下のプログラムを入力します。

```python
import M5

M5.begin()
print("Hello M5StickS3!")
```

`M5.begin()` は、M5StickS3の機能を初期化する処理です。

入力後、F5キーを押してプログラムを実行します。

Thonnyのシェルに以下のように表示されれば成功です。

```text
>>> %run -c $EDITOR_CONTENT
Hello M5StickS3!
>>>
```

これで、ThonnyからM5StickS3のMicroPythonプログラムを実行できるようになりました。

## 6. M5StickS3の画面に文字を表示する

次に、M5StickS3のディスプレイに文字を表示してみます。

以下のプログラムを入力します。

```python
import M5
import time

M5.begin()

M5.Lcd.fillScreen(0x000000)
M5.Lcd.setTextColor(0xFFFFFF)
M5.Lcd.setTextSize(2)

while True:
    M5.Lcd.fillScreen(0x000000)
    M5.Lcd.setCursor(10, 30)
    M5.Lcd.print("Hello!")
    M5.Lcd.setCursor(10, 60)
    M5.Lcd.print("M5StickS3")
    time.sleep(1)
```

F5キーを押して実行すると、M5StickS3の画面に「Hello!」「M5StickS3」と表示されます。

プログラムを停止する場合は、ThonnyのシェルでCtrl+Cを押します。

## 7. プログラムをM5StickS3に保存する

これまではThonnyからプログラムを実行していましたが、M5StickS3本体にプログラムを保存することもできます。

1. Thonnyの「ファイル」→「名前を付けて保存」を選択します。
2. 保存先に「MicroPythonデバイス」を選択します。
3. ファイル名を `main.py` として保存します。

これで、M5StickS3の内部ストレージにプログラムが保存されます。

ただし、**`main.py` を保存しただけでは、M5StickS3の電源投入時に自動実行されるとは限りません。**

今回の環境では、起動設定の変更が必要でした。

## 8. main.pyを自動実行する

M5StickS3の電源投入時に `main.py` を自動実行するため、M5Burnerで起動設定を変更します。

### 8.1 ダウンロードモードにする

1. Thonnyを終了します。
2. M5StickS3をUSBケーブルでパソコンに接続します。
3. 側面のリセットボタンを長押しし、内部の緑色LEDが点滅するダウンロードモードにします。

### 8.2 起動設定を変更する

1. M5Burnerを起動します。
2. StickS3のUIFlow2 V2.5.5を選択します。
3. `Configure` をクリックします。
4. `Boot Option` を `Run main.py directly` に変更します。
5. `Next` を押し、設定を書き込みます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1729073/f24e74be-a5c6-423a-b5c9-7d1294635f82.png)
**M5BurnerのBoot Option設定画面**

設定後、M5StickS3を再起動します。

今回の環境では、起動時に `main.py` が自動実行されることを確認できました。

これで、パソコンに接続しなくても、M5StickS3単体でプログラムを動作させることができます。

## 9. 使用時の注意点

### プログラムが実行できない場合

ThonnyでF5キーを押した際にUIFlow2の起動メッセージが表示され、プログラムが実行されない場合は、インタプリタの設定を確認してください。

「スクリプトを実行する前にインタプリタを再起動する」のチェックを外すことで、今回の環境では解決しました。

### main.pyが自動実行されない場合

M5BurnerのBoot Optionが `Run main.py directly` になっているか確認してください。

### main.pyを修正する場合

自動実行中のプログラムがThonnyからの操作を妨げる場合は、Ctrl+Cで実行を中断してから編集します。

また、プログラムの修正前には、パソコンにもバックアップを保存しておくと安心です。

## おわりに

今回は、M5StickS3にUIFlow2のファームウェアを書き込み、ThonnyでMicroPythonのプログラムを作成・実行するまでの手順を紹介しました。

さらに、プログラムを `main.py` として保存し、M5StickS3単体で自動実行できることも確認しました。

MicroPythonはPythonに似た文法で記述できるため、Arduino IDEで使用するC/C++に慣れていない人でも、比較的取り組みやすいと思います。

M5StickS3にはディスプレイやバッテリーが内蔵されているため、センサを接続することで、簡単な測定器として利用することもできます。

今後は、この環境を使って温湿度や振動などの測定にも取り組んでみたいと思います。