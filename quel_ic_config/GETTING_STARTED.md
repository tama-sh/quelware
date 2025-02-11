# quel_ic_config を使ってみる

quel_ic_config は、パッケージを作成し、それをpythonの任意の仮想環境にインストールして使用する。
以下では、仮想環境の作成から、簡単なコードの作成までをざっと説明する。

## 仮想環境の作成
パッケージの作成には[buildパッケージ](https://pypi.org/project/build/)を使う。
パッケージ作成する環境とパッケージをインストールする環境はおなじである必要はないが、ここでは、新しい仮想環境を作り、それを両方の用途で使用する。
任意の作業ディレクトリで以下の手順を実行すると、仮想環境が利用可能な状態になる。
```shell
python3.9 -m venv test_venv
source test_venv/bin/activate
pip install -U pip
```

## パッケージ作成とインストール
リポジトリ内の quel_ic_config ディレクトリに移動後、以下の手順で quel_ic_configのパッケージファイルを作成する。
```
pip install build
python -m build
```

パッケージファイルは、`dist/quel_ic_config-X.Y.Z-cp39-cp39-linux_x86_64.whl` (X,Y,Z は実際にはバージョン番号になる) という名前で作成される。
基本的にはこのファイルをインストールすればよいが、 pipで配布していない依存パッケージを先にインストールしておく必要がある。
現状、simplemulti版のファームウェアとfeedback版のファームウェアとで、一部の依存パッケージを挿し替える必要があることの注意が必要である。
simplemulti版のファームウェアを使うには、次のようにする。
```shell
pip install dependency_pkgs/*.whl simplemulti/*.whl
```

feedback版のファームウェアを使う場合には、次のようにする。
```shell
pip install dependency_pkgs/*.whl feedback/*.whl
```

最後に、本体パッケージのインストールを行う (X,Y,Z を適切なバージョン番号に置き換える。）
```shell
pip install dist/quel_ic_config-X.Y.Z-cp39-cp39-linux_x86_64.whl
```

なお、静的解析ツールやテストドライバなどの開発用のパッケージも一緒にインストールしたい場合には、次のようにすればよい。
```shell
pip install dependency_pkgs/extra/*.whl
pip install dist/quel_ic_config-X.Y.Z-cp39-cp39-linux_x86_64.whl\[dev\]
```

## シェルコマンドを使ってみる
quel_ic_config のパッケージにはいくつかの便利なシェルコマンドが入っており、仮想環境から使用できる。
仮に、10.1.0.42 のIPアドレスを持つ制御装置（QuEL-1 Type-A)をターゲットとして説明する。

### データリンク状態の確認
何か障害が発生したときに、まず装置のDAC/ADCのデータリンク状態を確認したくなるだろうと思う。
次のコマンドで状態を確認できる。

```shell
quel1_linkstatus  --ipaddr_wss 10.1.0.42 --boxtype quel1-a
```

装置が正常な運用状態にあれば、次のような出力を得る。
```text
AD9082-#0: healthy datalink  (linkstatus = 0xe0, error_flag = 0x01)
AD9082-#1: healthy datalink  (linkstatus = 0xe0, error_flag = 0x01)
```
`linkstatus` の正常値が0xe0であること、`error_flag`の正常値が0x01 であることを覚えておいて頂きたい。

初期化（いわゆるリンクアップ）が済んでいない場合には、次のような出力を得る。
```text
2023-11-07 17:54:29,111 [ERRO] quel_ic_config.ad9082_v106: Boot did not reach spot where it waits for application code
2023-11-07 17:54:29,111 [ERRO] quel_ic_config_utils.basic_scan_common: failed to establish a configuration link with AD9082-#0
2023-11-07 17:54:29,111 [ERRO] quel_ic_config_utils.basic_scan_common: AD9082-#0 is not working. it must be linked up in advance
2023-11-07 17:54:29,330 [ERRO] quel_ic_config.ad9082_v106: Boot did not reach spot where it waits for application code
2023-11-07 17:54:29,330 [ERRO] quel_ic_config_utils.basic_scan_common: failed to establish a configuration link with AD9082-#1
2023-11-07 17:54:29,330 [ERRO] quel_ic_config_utils.basic_scan_common: AD9082-#1 is not working. it must be linked up in advance
AD9082-#0: no datalink available  (linkstatus = 0x00, error_flag = 0x00)
AD9082-#1: no datalink available  (linkstatus = 0x00, error_flag = 0x00)
```
総合的な判定は、最後の2行にある。
正常な場合では0xe0だった`linkstatus` が、0x00 になっている。
ログ表示が少々鬱陶しいが、ログ情報はトラブルシューティングに有用なので敢えて表示している。

初期化に失敗するなどして、異常が発生している場合には、`link_status`が 0x90 や 0xa0 などの値となることが多い。
この場合には、再度リンクアップする必要がある。
また、内部のデータリンクにビットフリップが発生した場合には、`error_flag` が 0x11 になる。
量子実験の観点からは、直ちに問題が測定データに重篤な破損が出るわけではないが、折を見て再リンクアップすることが好ましい。
頻繁に再発する場合には連絡を頂きたい。

また、電源が入ってなかったり、ネットワークが繋がっていなかったりすると、以下のような出力が得られる。
```text
2023-11-07 17:35:01,753 [ERRO] quel_ic_config.exstickge_proxy: failed to send/receive a packet to/from ('10.5.0.42', 16384) due to time out.
2023-11-07 17:35:01,753 [ERRO] quel_ic_config_utils.basic_scan_common: failed to establish a configuration link with AD9082-#0
2023-11-07 17:35:01,753 [ERRO] quel_ic_config_utils.basic_scan_common: AD9082-#0 is not working. it must be linked up in advance
2023-11-07 17:35:02,254 [ERRO] quel_ic_config.exstickge_proxy: failed to send/receive a packet to/from ('10.5.0.42', 16384) due to time out.
2023-11-07 17:35:02,255 [ERRO] quel_ic_config_utils.basic_scan_common: failed to establish a configuration link with AD9082-#1
2023-11-07 17:35:02,255 [ERRO] quel_ic_config_utils.basic_scan_common: AD9082-#1 is not working. it must be linked up in advance
2023-11-07 17:35:02,756 [ERRO] quel_ic_config.exstickge_proxy: failed to send/receive a packet to/from ('10.5.0.42', 16384) due to time out.
AD9082-#0: failed to sync with the hardware due to API_CMS_ERROR_ERROR
2023-11-07 17:35:03,256 [ERRO] quel_ic_config.exstickge_proxy: failed to send/receive a packet to/from ('10.5.0.42', 16384) due to time out.
AD9082-#1: failed to sync with the hardware due to API_CMS_ERROR_ERROR
```
通信に失敗しているので、`linkstatus`の取得に失敗していることがログから読み取れる。

### 装置の再初期化（リンクアップ）
#### 基本的な使い方
次のコマンドで、指定の制御装置の初期化ができる。
```shell
quel1_linkup --ipaddr_wss 10.1.0.42 --boxtype quel1-a
```
初期化に成功した場合には、次のようなメッセージが表示される。
```text
ad9082-#0 linked up successfully
ad9082-#1 linked up successfully
```
この状態で、さきほどの `quel1_linkstatus` コマンドを使用すると正常状態を示す出力が得られるはずだ。

このコマンドは、デフォルトではType-AとType-Bの両方の機体で差し障りのないように、モニタ系が使用可能な状態に初期化を行う。
Type-Aの機体でリード系を使うように初期化をする場合には、次のようにする。
```shell
quel1_linkup  --ipaddr_wss 10.1.0.42 --boxtype quel1-a --config_options use_read_in_mxfe0,use_read_in_mxfe1
```
Type-Aのときにはリード系をデフォルトにすることも考えたのだが、敢えて愚直な作りにしている。

リンクアップの最中に次のような警告が何行か表示されることがあるが、最後に `linked up successfully` が出ていれば問題はない。
```text
2023-11-07 19:13:13,821 [WARN] quel_ic_config_utils.quel1_wave_subsystem: timeout happens at capture units CaptureUnit.U4, capture aborted
2023-11-07 19:13:22,718 [WARN] quel_ic_config.quel1_config_subsystem: failed to establish datalink between 10.5.0.42:AD9082-#1 and FPGA
```

リンクアップに失敗した場合には、警告が少なくとも10行程度表示された後に、次のようなエラーが出る。
```text
2023-11-07 19:54:28,830 [ERRO] root: ad9082-#0 failed to link up
2023-11-07 19:54:28,830 [ERRO] root: ad9082-#1 failed to link up
```
リンクアップが失敗を繰り返す場合には、警告ログの内容と共に連絡を頂きたい。
コマンドを`--verbose`オプション付きで実行すると、さらに詳細な情報が得られる。
このログを頂ければ、対応検討の返答までの時間の短縮が期待できる。

#### ハードウェアトラブルの一時的回避
`quel1_linkup`コマンドは、リンクアップ中に発生し得るハードウェア異常を厳格にチェックすることで、その後の動作の安定を担保するが、
時に部分的な異常を無視して、装置を応急的に使用可能状態に持ち込みたい場合には邪魔になる。
このような状況に対応するために、いくつかの異常を無視して、動作を続行するためのオプションを用意した。
あくまで応急的な問題の無視が目的なので、スクリプト内にハードコードして使うようなことは避けるべきである。

- CRCエラー
  - FPGAから送られてきた波形データの破損をAD9082が検出したことを示す。`(linkstatus = 0xe0, error_flag = 0x11)` でリンクアップが失敗することでCRCエラーの発生が分かる。
  - `--ignore_crc_error_of_mxfe` にCRCエラーを無視したい MxFE (AD9082) のIDを与える。カンマで区切って複数のAD9082のIDを与えることもできる。
- ミキサの起動不良
  - QuBEの一部の機体において、電源投入時にミキサ(ADRF6780)の起動がうまく行かない事象が知られている。`unexpected chip revision of ADRF6780[n]`(n は0~7) で通信がうまく行っていないミキサのIDが分かる。   
  - `--ignore_access_failure_of_adrf6780` にエラーを無視したいミキサのID (上述のn)を与える。

これらの一部は他のコマンドと共通である。他のコマンドへの適用可否については、各コマンドの`--help`を参照されたい。
これら以外にも、`ignore`系の引数がいくつかあるが、開発目的のものである。

#### ADCの背景ノイズチェックについて
リンクアップ手順の最後に、各ADCについて、無信号状態での読み値に大きなノイズが乗っていないことの確認をしている。
QuEL-1の各個体については、キュエル株式会社がノイズが既定値(=256)よりも十分に小さいことを確認後に出荷しているが、QuBEについてはノイズが大きめの機体が存在する。

```text
max amplitude of capture data is XXXXX (>= 256), failed to linkup
```

のようなメッセージが5回以上繰り返し出て、リンクアップに失敗する場合には、`--background_noise_threshold` 引数で上限値を引き上げられる。
目安としては、400くらいまでが個体差の範疇であると考えてよい。
それよりも大きい値で失敗を繰り返す場合には、なんらかのハードウェア的な問題を示唆するので、装置のパワーサイクルを行い、それでも回復しない場合にはサポート窓口へ連絡して頂きたい。

なお、リンクアップ中に30000以上の値が数回出た後に、リンクアップに成功するのは既知の事象であり、装置の使用上問題はない。
初期化時のタイミングに依存した異常で、確率的に発生するが、一旦、異常なしの状態に持ち込めば、その後は安定動作する。
なお、`quel1_linkup`コマンドは異常が発生しなくなるまでリンクアップを自動で繰り返す。

#### 分かりにくい警告メッセージについて 
QuBE-RIKENの制御装置を使用する際に `--boxtype` を `quel1-a` や `quel1-b` と誤って指定した後に、正しく `qube-riken-a` や `qube-riken-b` に
指定し直した場合に、次のような警告メッセージが出ることがある。
```
invalid state of RF switch for loopback, considered as inside
```

`quel1_linkup` でこのメッセージが出る場合、つまり、当該装置を間違った boxtype で使用した後で、正しい boxtype で再リンクアップを試みた場合には、無害なので無視してよい。
なぜならば、`quel1_linkup`コマンドがスイッチの状態を初期化するからである。
他の状況でこのメッセージが出る場合には、すべてのRFスイッチの状態を再設定しておいた方がよいだろう。


### 装置の設定状態の確認
各ポートの設定パラメタの一覧を次のコマンドで確認できる。
```shell
quel1_dump_port_config --ipaddr_wss 10.1.0.42 --boxtype quel1-a
```

全てのポートのについて、次のような情報が表示される。
```text
{'group-#0': {'channel_interporation_rate': 4, 'main_interporation_rate': 6},
 'group-#1': {'channel_interporation_rate': 4, 'main_interporation_rate': 6},
 'port-#00': {'direction': 'in',
              'rfswitch': 'looping back',
              'lo_hz': 11000000000,
              'cnco_hz': 1500000000.0,
              'channels': [{'fnco_hz': 0.0}]},
 'port-#01': {'direction': 'out',
              'channels': [{'fnco_hz': 0.0}],
              'cnco_hz': 1500000000.0
              'fsc_ua': 40527,
              'lo_hz': 11000000000,
              'sideband': 'L',
              'rfswitch': 'blocked'},
 'port-#02': {'direction': 'out',
              'channels': [{'fnco_hz': 0.0},
                           {'fnco_hz': -100000000.0},
                           {'fnco_hz': 100000000.0}],
              'cnco_hz': 1500000000.0,
              'fsc_ua': 40527,
              'lo_hz': 8500000000,
              'sideband': 'U',
              'rfswitch': 'blocked'},
...,
}
```

ただし、取得する方法がない出力ポートのVATTの値だけは表示されないので注意が必要である。
（このコマンドの実装に用いているAPIは、おなじ実行コンテキストで設定されたVATT値をキャッシュして返すが、コマンド自身がVATTの値を設定する
ことはないので、何も値が得られない。）

## APIを使ってみる
次のコマンドで、装置の簡単な動作確認をインタラクティブに実施できる。
あくまでお試し用なので複雑なことをするのには向いていない。
リポジトリの`quel_ic_config`ディレクトリに移動後、次のコマンドをすることで制御装置を抽象化したオブジェクト`box`が利用可能な状態になった
pythonのインタラクティブシェルが得られる。

```shell
python -i scripts/getting_started_example.py --ipaddr_wss 10.1.0.42 --boxtype quel1-a 
```

たとえば、`box.dump_config()` とすることで、上述の `quel1_dump_port_config` コマンドの出力同様の結果を含んだデータ構造を得られる。
以下に APIの一覧をアルファベット順に列挙する。
詳しい使い方は、`help(box.dump_config)` のように `help`関数で見られたい。

- activate_monitor_loop (モニタ系のRFスイッチをループバック状態にする。)
- activate_read_loop  (リード系のRFスイッチをループバック状態にする。)
- close_rfswitch  (指定のポートのRFスイッチを信号をポートから出さない状態にする。）
- config_channel  (指定のポートのチャネライザの設定をする。FNCO周波数の設定であると考えてよい。)
- config_port  (指定のポートのパラメタの設定をする。)
- deactivate_monitor_loop  (モニタ系のRFスイッチのループバックを解除する。)
- deactivate_read_loop  (リード系のRFスイッチのループバックを解除する。)
- dump_config　　（各ポートの設定状態をデータ構造として得る。）
- easy_capture  (指定の入力ポートの信号をキャプチャする最も簡単な方法。)
- easy_start_cw  (指定の出力ポートから信号を出力する最も簡単な方法。）
- easy_stop  (指定の出力ポートの信号を停止する。)
- easy_stop_all  (全ての出力ポートの信号を停止する。)
- is_loopedback_monitor  (モニタ系のループバックの状態を返す。True ならループバック状態。)
- is_loopedback_read  (リード系のループバックの状態を開けす。True ならループバック状態。)
- open_rfswitch  (指定のポートのRFスイッチを信号をポートから出す状態にする。）
- start_channel  (指定のポートの指定のチャネライザで、CWの発生を開始する。）
- start_channel_with_wave  (指定のポートの指定のチャネライザで、指定の波形データの波形発生を開始する。)
- stop_channel  (指定のポートの指定のチャネライザからの波形発生を停止する。）

`easy_start` は `config_port`, `config_channel`, `open_rfswitch` 及び `start_channel` を適切な順番で呼び出している。
そのレベルでの詳細は、ソースコード`quel_ic_config_util/simple_box.py` あたりを見られたい。

## 次のステップ
`scripts`ディレクトリ内にある`getting_started_example.py` 以外にもサンプルコードについて紹介しておく。
定数がハードコートされているなど、使い勝手にやや難はあるが、参考になると思う。

- `manual_test_code_prebox.py`: SimpleBoxオブジェクトの内部のオブジェクトを使った各種実行テストをするためのコンソール。
- `simple_scheduled_loopback.py`: 内部のカウンタを使って、一定間隔で5回、キャプチャを繰り返す。対象機体（10.1.0.74がハードコートされている）の2つのモニタアウトをコンバイナを介して、グループ0のRead-inに繋いだ状態で使う。 
- `twobox_scheduled_loopback.py`: 内部のカウンタを使って、2台の制御装置（10.1.0.74 と 10.1.0.58) を同期してキャプチャする。2台の制御装置の合計4つのモニタアウトをコンバイナを介して、1台目のグループ0のRead-inに繋いだ状態で使う。
- `skew_scan.py`: `twobox_schedued_loopback.py`の内容を、開始タイミングを1クロックずつずらしながら17回行い、波形生成のタイミング関係に与える影響を確認するサンプル。

なお、これらの実装の一部は、近い将来に公開される機能のテストベッドも含む。
