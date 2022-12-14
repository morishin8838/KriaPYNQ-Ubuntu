# HDMI-IPを組み込んだDPU-PYNQプラットフォームの組込み手順
* Xilinx HDMI-IPを組み込んだ回路にDPU-PYNQを組み込む場合の手順です。
* HDMI-IPを組み込まない場合等、リソースに余裕がある場合、本手順にする必要はなし

## 注意事項
* ここでは、DPUサイズはB3136を用います。 Vitis-AIで配布される推論ファイルはB4096サイズです。
* B3136サイズの回路で、B4096の推論ファイルを用いるとクラッシュします。注意してください。私はこの問題が気が付かず、１ヶ月悩みました。
* B3136 DPU-IP用の推論ファイル作成方法は、別に手順を示す。
## 回路設計 Vivado
1. Xilinx HDMI-IPがあるK26 FPGAカスタム回路設計を行う
2. Clock追加
    * 200MHz,400MHz出力と、それぞれ200M用、400M用のSysReset追加する。
    * 300MHz,600MHz設定のメイクして、TimingErrorが発生した場合のみ200M/400Mに落とす手順でもよい（プラットフォームクロック再設定忘れずに）
3. plartform設定
    * AXI port
        * M_HPM1_FPD:MLPD
        * S_HPC1_FPD:HPC1
        * S_HP1_FPD:HP1
    * Clock
        * pl_clk0:Enabledチェック, is Defaultチェック
        * clk_out1:100MHz Enabledチェック、id=任意
        * clk_out2:200MHz Enabledチェック、id=2
        * clk_out3:300MHz Enabledチェック、id=任意
        * clk_out4:400MHz Enabledチェック、id=4     
        
4. bitstream生成
5. xsaファイルをエクスポート生成

## プラットフォーム作成 Vitis
1. vitis起動し、メニューからプラットフォーム選択し、プラットフォームを生成する。
2. エクスポートしたxsaファイルを選択し、linuxを選択し、bootはチェック外す。ワーニングは無視して、終了
3. トンカチボタンでbuildする。
4. exportディレクトリ配下にあるxpfmファイルのパスを確認する。下記、VITIS_PLATFORMのパスに指定する

## ビルド DPU-PYNQ
0. 下準備
    1. ビルド環境をダウンロードする
        * $wget https://github.com/Xilinx/DPU-PYNQ
        * $cd board
    2. 評価ボード(ZCU104,Ultra96,KV260)を使う場合、このままでよい
    3. カスタムボードを使う場合、適当なディレクトリを作成する。
        * 作成したディレクトリにdpu_conf.vh,prj_configを配置する。仕様が似ている評価ボードの設定ファイルをコピーすること
        * dpu_conf.vh,prj_configの編集内容は、設定ファイル編集を参照する。
1. 設定ファイル編集 <メイクroot>/boards/ボード名/dpu_conf.vh
    1. dpu_conf.vh編集
        * `define B3136
        * `define def_UBANK_IMG_N          4
        * `define def_UBANK_WGT_N          15
        * `define def_UBANK_BIAS           1
    2. prj_config
        * freqHz=200000000:DPUCZDX8G_1.aclk
        * freqHz=400000000:DPUCZDX8G_1.ap_clk_2
        * id=2:DPUCZDX8G_1.aclk
        * id=4:DPUCZDX8G_1.ap_clk_2
        * sp=DPUCZDX8G_1.M_AXI_GP0:HPC1
        * sp=DPUCZDX8G_1.M_AXI_HP0:HP1
        * sp=DPUCZDX8G_1.M_AXI_HP2:HP1
        * prop=run.impl_1.strategy=Performance_Explore
    
2. ビルド実行  
$VERSION=2021.1   
$source /tools/Xilinx/Vitis/$VERSION/settings64.sh  
$source /opt/xilinx/xrt/$VERSION/setup.sh  
$make clean  
$make BOARD=NKV VITIS_PLATFORM=<path xpfm>/xxxxx.xpfm  

* CPUパワーにもよりますが、ビルドは3時間コースです。


## 注意事項 HDMI-IP特有問題
1. bitファイルを再度に生成したい場合、ビルドしたVivadoプロジェクトをVivadoを用いて、設定変更し再bitstream生成することが可能
   * <Make用 PATH>/boards/NKV/binary_container_1/link/vivado/vpl/prj/prj.xpr
2. HDMI-IP利用の際、"Channel primitive pin location constraints"がビルド際に原因不明で指定値に制御できないため、手動で再bitstream生成で対応した。
   * Vivadoからprj.xprを開き、synthesis実行後、IOプラニングからY2<=>T2を逆にし、手動でimpl,bitstream生成を行う。
   * #set_property package_pin Y2 [get_ports gthrxp_in[2]]
   * #set_property package_pin T2 [get_ports gthrxp_in[0]]
