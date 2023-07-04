# FasterR-CNN
## 概要
- Fast R-CNNはDeep-LearningによるEndo-to-Endoの学習に初めて成功した物体検出モデル
- RPN(Region Proposal Network)と呼ばれるCNN構造を用い、ROI抽出を深層学習に行わせるよう改良を行った
- これでボトルネックとなっていたSelective Search処理が不要となり、大幅に精度と検知スピードが向上した

## アーキテクチャ
<img alt="FasterR-CNN" src=./image/faster-rcnn.png></img>
- 処理は以下
1. 入力画像から特徴マップを出力する処理（学習済みVGG16などを流用）
2. RPNと呼ばれる物体が写っている場所と、その矩形の形を得る処理
3. 2で抽出したROIと呼ばれる入力を固定長に変換する処理
4. 何が写っているかを判断する分類処理

#### 1, 特徴量マップを出力
- 分類のタスク用に事前トレーニングされたCNNモデル（VGG16やResNet等）を使用し、中間層の出力を特徴マップを作成
- 特徴マップのサイズは、VGG16の場合、４回プーリングを実施しているため、元の画像の１/16の縦横をもった512次元のデータに。元の画像が300×400×3であれば、特徴マップは18×25×512になる
<img alt="FasterR-CNN" src=./image/base-cnn.png></img>

#### 2, RPNで物体の位置を検出
- RPNは「input画像中で、どこに物体が写っているのか（物体が写っている場所と、その矩形の形）」を検出する機械学習モデル。この時、**何が写っているかまではRPNでは判断しない**

##### 1) AnchorとAnchor boxesの設定
- 特徴マップを生成したら次にAnchorを設定する。Anchorは特徴マップ上の各点になり、それぞれのAnchorに対して９つ（ハイパーパラメータで指定）のAnchor boxesを作成する
- 各Ancho boxesと正解（Ground Truth）を見比べた情報がRPNを学習させる際の教師データとなる
- Anchorは矩形を描く際の中心となります。特徴マップが18×25×512の場合、18×25=450個のAnchorが設定され、下図のように等間隔に設定される
<img alt="FasterR-CNN" src=./image/anchor.png></img>
- Anchorは等間隔に設定されるため、「物体」がどこにあってもどれかしらのAnchorがその「物体」の中心となることが可能（＝矩形の中心候補）
- 物体の中心位置が決まるとあとは矩形の縦横が決まれば物体の検出枠が描画できますが、この矩形の縦横を決めるのがAnchor box
- 各Anchorから「基準の長さ」と「縦横比」をそれぞれ決めることで、複数のAnchor boxesを作り出します。例えば、基準の長さ=(64, 128, 256)、縦横比=(1:1, 1:2, 2:1)とすると、下図5のようなAnchor boxesが生成される
- このとき１つのAnchorに対して、基準の長さ(3要素) ×　縦横比(3要素)=9個の矩形が生成される
- また、各Anchor boxesの面積は揃える必要がある（ex.基準の長さ=64であれば、縦横比＝1:1のときは64×64=4096なので縦横比=1:2のときは縦45×横91(=4096)の矩形、縦横比=2:1のときは縦91×横45(=4096)の矩形）。
- 実際には画像からはみ出したAnchor boxesは無視され、実際は以下のようになる
<img alt="FasterR-CNN" src=./image/anchor-re.png></img>
- Anchor boxesはVGG16、画像300×400×3を使用すれば、最大で18×25×9=4050個作られる。ここで様々な形のbox(矩形)が提案されることにより、正解（ground truth）がどんな形であっても正解と似ているboxの候補を生成することができる
- 作成したAnchor boxと正解（ground truth）を見比べて、その結果をRPNの出力とする

##### 2) RPN層での出力
- Anchor boxの中身が背景か物体か（下図のcls layerの2k scores）、物体だった場合は正解（ground truth）とどのくらいズレているのか（下図のreg layerの4k coordinates）」の２項目を出力
- 「k」はAnchor boxesの数
<img alt="FasterR-CNN" src=./image/rpn.png></img>
- 「背景が物体か」については、ground truthとAnchor boxesのIoUを計算し、「IoU < 0.3」なら「背景」、「IoU > 0.7」なら「物体」とラベルを付ける
- そのため、9(Anchor boxesの数) × 2（ラベル） = 18クラスが作られます。なお、「0.3 ≦ IoU ≦ 0.7」のboxについては、背景とも物体とも言えないので学習には使用しない（IoUの閾値もハイパーパラメータで指定）
<img alt="FasterR-CNN" src=./image/label.png></img>
- 「ground truthであるAnchor boxのズレ」については、「中心ｘ座標のズレ」、「中心y座標のズレ」、「横の長さのズレ」、「縦の長さのズレ」という４つの指標で評価。そのため、9(Anchor boxesの数) × 4（ズレ) = 36クラスが作成される
<img alt="FasterR-CNN" src=./image/reg.png></img>

#### 3, クラス分類と矩形の推定
- 最後にPRNによって抽出された領域をVGG16等の出力である特徴量マップにマッピング。ROIプーリングによって固定長のベクトルに変換し、全結合層でクラス分類と矩形推定を行う

## 学習
- RPN部分の学習とFaster R-CNN全体の学習を交互に行い、それぞれに精度を高めながら学習を進める

#### 1, RPNの学習
1. 入力画像をVGG16等に通して特徴マップを得る
2. 1に対し、さらに３×３の畳み込み（512チャネル）を行い、さらに1×1の畳み込みをかけることで、「背景か物体か」用のアウトプットと、「ground truthとのズレ」用のアウトプットの２つを作る
-「物体か背景か」の２値分類問題と、「ground truthとのズレ」の回帰問題を同時に解く。**前者はバイナリクロスエントロピー、後者はＬ１ノルム（絶対値誤差）をベースにした誤差関数**を適用して学習を行う

#### 2, Faster R-CNN全体の学習
1. 入力画像をVGG16等に通して特徴マップを得る（RPNでの学習と同じもの）
2. ROI Poolingを使用し、特徴マップを固定長のサイズにする
3. 2で得たベクトルを4096の全結合層に2回通し、最後にクラス分類用と、矩形ズレ回帰用の２種類の出力を得る
- RPNを学習させるために用いた「3×3の畳み込み ＋ 1×1の畳み込み」は、RPN特有の構造のため、Faster R-CNNで何かを予測する際には使用しない
- 3の全結合層部分の入出力は **常に固定サイズとなるため、** Faster R-CNNでは入力画像のサイズに依存しない学習、予測が可能になる

## 参考：
1. https://qiita.com/DeepTama/items/0cb9ca2d35c200deed37
2. https://medium.com/lsc-psd/faster-r-cnn%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8Brpn%E3%81%AE%E4%B8%96%E7%95%8C%E4%B8%80%E5%88%86%E3%81%8B%E3%82%8A%E3%82%84%E3%81%99%E3%81%84%E8%A7%A3%E8%AA%AC-dfc0c293cb69
3. https://github.com/Yagami360/machine-learning-papers-survey/issues/76
