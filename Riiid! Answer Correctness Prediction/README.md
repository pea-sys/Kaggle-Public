■成績  
94 / 3,395 teams(ソロ銀メダル)

■コンペティションホスト  
韓国のAIスタートアップ企業。
TOEICの学習アプリを提供しています。

■概要  
アプリで出題した問題を各ユーザーが正解するか予測します。  
コードコンペティションです。  
Kaggleが提供するKernel上で推論プログラムを動作させる必要があります(推論時はインターネット使用不可)。  
ただし、モデルはローカルで訓練可能。  
テストセットは、Kaggleが提供するTimeSeriesAPIを使用して動的に与えられます。

■評価指標  
AUC

■データセット  
[こちら](https://www.kaggle.com/c/riiid-test-answer-prediction/data)
※データサイズが大きいのでメモリ管理が重要になります


## ■行動履歴(時系列順)  
**1.TimeSeriesAPIとの格闘**  
TimeSeriesAPIの扱いやテストセットに対する考慮不足により、  
  ひたすら投稿エラーを繰り返していました(テストセットのハック対策のためか、  
  具体的なエラー内容は通知されません)。  
  徐々にディスカッションで問題解決方法が共有されるようになり、  
  解決できるようになりました。  
  * 投稿に成功するためには、1イテレーションあたり0.55sec以内(pandasのmergeを使用するだけでタイムアウトになります)  
  * 未知のユーザー等の存在考慮
  
  
**2.オンラインラーニング**  
Increment Learningの綺麗な[Kernel](https://www.kaggle.com/spacelx/2020-r3id-incremental-learning-pytorch-creme)が公開されていました  
これまで触れる機会がなかったので、勉強がてら2週間程このモデルをベースに改良を続けてました。  
LightGBMの[オンラインラーニング](https://gist.github.com/goraj/6df8f22a49534e042804a299d81bf2d6)も試したりしてました。  
[Datatableパッケージを使用したIncrementalモデル](https://www.kaggle.com/rohanrao/riiid-ftrl-ftw)も試しました。  
DatatableはRの高速データ読み取りパッケージです。爆速でRAM消費も少なく感激しました。  
CPUのキャッシュを積極的に使ってるのかもしれません。  

ただ、訓練データ1億レコードに対して、テストデータは250万レコードなので  
オンラインラーニングのアルゴリズムで精度を出すこと難しかったです。  
Stackingに使っても良かったかもしれません。  
DisucussionでCremeの作者もオンラインラーニングではいい結果は得られないとコメントしてました  

**3.LightGBMと速度とメモリ対策**  
結局LightGBMに帰ってきました。  
[ベースライン](https://www.kaggle.com/its7171/lgbm-with-loop-feature-engineering)をフォークし、これをベースに締切4日前まで色々な特徴を作成したり、削ったりしていました
コンペ終了後の数チームのLightGBMの解法を見ましたが、効く特徴は自力でほぼ全部作れていたので良かったです。実のところ、EDAはほとんど行ってません(メモリが厳しい)。 
自分が予測する立場だったら、こういった情報が欲しいなと思ったものを追加していきました。  
実験回数を増やすのと推論時間の制限を恐れなくていいようにCythonの書籍を購入・学習し、ループのCython化を進め高速化を行いました。 
ループの並列化はGILを外す必要があり、結構大変そうだったのでやめました。  
メモリ対策としては、ビットあたりの情報量を増やすために、2値しかもたない類似の特徴量を合体させたり、カテゴリ値をサンプリングすることでデータサイズを減らす工夫を行いました。  
無料ユーザのレコードの学習時の重みを下げたりしましたが、効きませんでした。

## ■感想
計画通り、巨大データサイズを扱ったコンペでKernelのみ利用して銀メダルが取得できて良かったです。  
KernelはRAM16GBのため、ちょっと効く位の特徴は入れる隙間がありませんでした。  
なので、弱い特徴をより強い特徴に入れ替えていく作業が中々しんどかったです。  
上位は強力なNN(SaintやSAKT)を使用してました。Transformer位は想像力で組めるようになる必要があるなと感じました。  
今回はCython.Incremental learning, datatable等、色々学びが多くて良かったです。  
NN系のソリューションはこれから読んで、理解しながら動かしていく予定です。  
モデルのパラメータチューニングは時間が足りなくて1回しかやってないです。  


最終的なプログラム構成  
--Model Deploy Notebook*3(訓練データが異なるモデル3つを生成)  
--Data Deploy Notebook(全データを集計したデータ生成。訓練に全データは使用できなかったため)  
--Data Deploy2 Notebook(Datatableを用いて集計したデータ生成)  
--Predictor Notebook(上記Notebookの出力をinputとして、モデルアンサンブルして推論。モデルの重みはTimeSeriesAPIから帰ってくる回答とのlossを加算し、一番少ないものの重みを増やす)  

メモ：
pandasはもう並列化されてる処理が多いせいかpandarallelは利かなかった  
pandasのmap関数での辞書適用は稀に10倍程度遅い時があった  
LightGBMはDataset作成時に渡されたデータをまとめてnumpy.float32にキャストするため、メモリスパイクが発生する。事前にfor文で１つずつキャストすることでメモリスパイクを抑えることが出来る  
LightGBMのカテゴリ特徴量にハイカーディナルな特徴量を設定するとメモリ消費が大幅に増える  
