# 注意機構とトランスフォーマー

## [講義前のクイズ](https://red-field-0a6ddfd03.1.azurestaticapps.net/quiz/118)

NLP分野で最も重要な問題の一つは、**機械翻訳**であり、これはGoogle翻訳などのツールの基盤となる重要なタスクです。このセクションでは、機械翻訳、またはより一般的には*シーケンスからシーケンス*のタスク（これは**文の変換**とも呼ばれます）に焦点を当てます。

RNNを使用したシーケンスからシーケンスの実装は、二つの再帰的ネットワークによって行われます。一方のネットワーク、すなわち**エンコーダ**は、入力シーケンスを隠れ状態に圧縮し、もう一方のネットワーク、すなわち**デコーダ**は、この隠れ状態を翻訳結果に展開します。このアプローチにはいくつかの問題があります：

* エンコーダネットワークの最終状態は文の最初を記憶するのが難しく、長い文に対してモデルの質が低下します。
* シーケンス内のすべての単語が結果に同じ影響を与えます。しかし、実際には入力シーケンス内の特定の単語が他の単語よりも順次出力に対してより大きな影響を持つことがよくあります。

**注意機構**は、RNNの各出力予測に対する各入力ベクトルの文脈的影響を重み付けする手段を提供します。これは、入力RNNの中間状態と出力RNNの間にショートカットを作成することによって実装されます。この方法では、出力シンボルy<sub>t</sub>を生成する際に、異なる重み係数α<sub>t,i</sub>を持つすべての入力隠れ状態h<sub>i</sub>を考慮します。

![加法注意層を持つエンコーダ/デコーダモデルの画像](../../../../../translated_images/encoder-decoder-attention.7a726296894fb567aa2898c94b17b3289087f6705c11907df8301df9e5eeb3de.ja.png)

> [Bahdanau et al., 2015](https://arxiv.org/pdf/1409.0473.pdf)の加法注意機構を持つエンコーダ-デコーダモデル、[このブログ投稿](https://lilianweng.github.io/lil-log/2018/06/24/attention-attention.html)から引用

注意行列 {α<sub>i,j</sub>} は、特定の入力単語が出力シーケンス内の特定の単語の生成にどの程度寄与しているかを表します。以下はそのような行列の例です：

![RNNsearch-50によって見つかったサンプルアラインメントの画像、Bahdanau - arviz.orgから](../../../../../translated_images/bahdanau-fig3.09ba2d37f202a6af11de6c82d2d197830ba5f4528d9ea430eb65fd3a75065973.ja.png)

> [Bahdanau et al., 2015](https://arxiv.org/pdf/1409.0473.pdf)からの図（Fig.3）

注意機構は、現在または近い将来のNLPにおける最先端技術の多くを担っています。しかし、注意を追加するとモデルのパラメータ数が大幅に増加し、RNNのスケーリング問題を引き起こしました。RNNをスケーリングする際の重要な制約は、モデルの再帰的性質がトレーニングのバッチ処理と並列化を難しくすることです。RNNでは、シーケンスの各要素は順次の順序で処理する必要があり、簡単には並列化できません。

![注意付きエンコーダデコーダ](../../../../../lessons/5-NLP/18-Transformers/images/EncDecAttention.gif)

> [Googleのブログ](https://research.googleblog.com/2016/09/a-neural-network-for-machine.html)からの図

注意機構の採用とこの制約の組み合わせにより、現在私たちが知っていて使用している最先端のトランスフォーマーモデル（BERTからOpen-GPT3まで）が作成されました。

## トランスフォーマーモデル

トランスフォーマーの背後にある主なアイデアの一つは、RNNの順次性を回避し、トレーニング中に並列化可能なモデルを作成することです。これは、二つのアイデアを実装することによって達成されます：

* 位置エンコーディング
* RNN（またはCNN）ではなく自己注意機構を使用してパターンをキャプチャする（これが、トランスフォーマーを紹介する論文が* [Attention is all you need](https://arxiv.org/abs/1706.03762)と呼ばれる理由です）

### 位置エンコーディング/エンベディング

位置エンコーディングのアイデアは次のとおりです。
1. RNNを使用する際、トークンの相対位置はステップ数によって表され、明示的に表現する必要はありません。
2. しかし、注意に切り替えると、シーケンス内のトークンの相対位置を知る必要があります。
3. 位置エンコーディングを取得するために、トークンのシーケンスにシーケンス内のトークン位置のシーケンス（すなわち、0,1,...の数字のシーケンス）を追加します。
4. 次に、トークン位置をトークンエンベディングベクトルと混ぜます。位置（整数）をベクトルに変換するために、さまざまなアプローチを使用できます：

* トークンエンベディングに類似した学習可能なエンベディング。これがここで考慮するアプローチです。トークンとその位置の両方にエンベディング層を適用し、同じ次元のエンベディングベクトルを得て、それらを加算します。
* 元の論文で提案された固定位置エンコーディング関数。

<img src="images/pos-embedding.png" width="50%"/>

> 著者による画像

位置エンベディングを使用すると、元のトークンとシーケンス内のその位置の両方が埋め込まれます。

### マルチヘッド自己注意

次に、シーケンス内のパターンをキャプチャする必要があります。これを行うために、トランスフォーマーは**自己注意**機構を使用します。これは基本的に、同じシーケンスに対して入力と出力に適用される注意です。自己注意を適用することで、文内の**コンテキスト**を考慮し、どの単語が相互関連しているかを確認できます。例えば、*it*のようなコリファレンスによって参照される単語を確認し、コンテキストも考慮に入れることができます：

![](../../../../../translated_images/CoreferenceResolution.861924d6d384a7d68d8d0039d06a71a151f18a796b8b1330239d3590bd4947eb.ja.png)

> [Googleブログ](https://research.googleblog.com/2017/08/transformer-novel-neural-network.html)からの画像

トランスフォーマーでは、**マルチヘッド注意**を使用して、ネットワークがさまざまなタイプの依存関係（例えば、長期的対短期的な単語関係、コリファレンス対他の何かなど）をキャプチャできるようにします。

[TensorFlowノートブック](../../../../../lessons/5-NLP/18-Transformers/TransformersTF.ipynb)には、トランスフォーマー層の実装に関する詳細が含まれています。

### エンコーダ-デコーダ注意

トランスフォーマーでは、注意は二つの場所で使用されます：

* 自己注意を使用して入力テキスト内のパターンをキャプチャする
* シーケンス翻訳を実行する - エンコーダとデコーダの間の注意層です。

エンコーダ-デコーダ注意は、RNNで使用される注意機構と非常に似ています。このセクションの冒頭で説明したように。このアニメーション図は、エンコーダ-デコーダ注意の役割を説明しています。

![トランスフォーマーモデルで評価がどのように行われるかを示すアニメーションGIF](../../../../../lessons/5-NLP/18-Transformers/images/transformer-animated-explanation.gif)

各入力位置が各出力位置に独立してマッピングされるため、トランスフォーマーはRNNよりも良好に並列化でき、はるかに大きく表現力豊かな言語モデルを可能にします。各注意ヘッドは、単語間の異なる関係を学習するために使用でき、下流の自然言語処理タスクを改善します。

## BERT

**BERT**（Bidirectional Encoder Representations from Transformers）は、*BERT-base*用の12層、*BERT-large*用の24層を持つ非常に大きなマルチレイヤートランスフォーマーネットワークです。このモデルは、無監督トレーニング（文中のマスクされた単語を予測）を使用して、大規模なテキストデータコーパス（WikiPedia + 書籍）で事前トレーニングされます。事前トレーニング中に、モデルは言語理解の重要なレベルを吸収し、その後ファインチューニングを使用して他のデータセットと活用できます。このプロセスは**転移学習**と呼ばれます。

![http://jalammar.github.io/illustrated-bert/からの画像](../../../../../translated_images/jalammarBERT-language-modeling-masked-lm.34f113ea5fec4362e39ee4381aab7cad06b5465a0b5f053a0f2aa05fbe14e746.ja.png)

> 画像 [出典](http://jalammar.github.io/illustrated-bert/)

## ✍️ 演習：トランスフォーマー

次のノートブックで学習を続けてください：

* [PyTorchでのトランスフォーマー](../../../../../lessons/5-NLP/18-Transformers/TransformersPyTorch.ipynb)
* [TensorFlowでのトランスフォーマー](../../../../../lessons/5-NLP/18-Transformers/TransformersTF.ipynb)

## 結論

このレッスンでは、トランスフォーマーと注意機構について学びました。これらはすべてNLPツールボックスの重要なツールです。BERT、DistilBERT、BigBird、OpenGPT3など、ファインチューニング可能なトランスフォーマーアーキテクチャの多くのバリエーションがあります。[HuggingFaceパッケージ](https://github.com/huggingface/)は、これらのアーキテクチャの多くをPyTorchとTensorFlowの両方でトレーニングするためのリポジトリを提供しています。

## 🚀 チャレンジ

## [講義後のクイズ](https://red-field-0a6ddfd03.1.azurestaticapps.net/quiz/218)

## レビュー & 自習

* トランスフォーマーに関する古典的な[Attention is all you need](https://arxiv.org/abs/1706.03762)論文を説明する[ブログ投稿](https://mchromiak.github.io/articles/2017/Sep/12/Transformer-Attention-is-all-you-need/)。
* トランスフォーマーに関する一連のブログ投稿で、アーキテクチャの詳細を説明しています。[詳細はこちら](https://towardsdatascience.com/transformers-explained-visually-part-1-overview-of-functionality-95a6dd460452)。

## [課題](assignment.md)

**免責事項**：  
この文書は、機械ベースのAI翻訳サービスを使用して翻訳されています。正確性を追求していますが、自動翻訳には誤りや不正確さが含まれる可能性があることにご留意ください。元の文書はその母国語で権威ある情報源と見なされるべきです。重要な情報については、専門の人間翻訳をお勧めします。この翻訳の使用によって生じる誤解や誤訳について、当社は一切の責任を負いません。