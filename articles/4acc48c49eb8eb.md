---
title: "【自然言語処理】BERTの単語ベクトルで「王+女-男」を計算してみる"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["自然言語処理", "faiss", "python", "ベクトル", "bert"]
published: true
---

ベクトルの近傍探索ライブラリfaissの操作備忘録を書きたかったのですが、それだけだとつまらなかったので、Word2Vec等で有名な単語ベクトルの演算がBERTにより獲得されたベクトルでもできるのか調べてみました。

# 事前準備

::: details ライブラリのインストール
```shell
python3 -m venv .env
source .env/bin/activate

pip install faiss-cpu transformers numpy
```
:::

::: details BERTの静的単語ベクトルの取り出し
```python
import torch
from transformers import AutoModel, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("cl-tohoku/bert-base-japanese-whole-word-masking")
model = AutoModel.from_pretrained('cl-tohoku/bert-base-japanese-whole-word-masking')

token_embeds = model.get_input_embeddings().weight.clone()
vocab = tokenizer.get_vocab()
vectors = {}

for idx in vocab.values():
    vectors[idx] = token_embeds[idx].detach().numpy().copy()
```
:::

::: details BERTの動的単語ベクトルの取り出し
いわゆるcontextualized word representaionと呼ばれるやつで、出力ベクトルにあたります。
文脈まで考えると大変なことになるので、今回使用するモデルの32000個の語彙を[CLS]と[SEP]トークンで挟んで入力してベクトル表現を獲得したいと思います。tokenizerにそのまま突っ込めるとよかったのですが、サブワードがらみの仕様が面倒だったのでそのまま単語IDを入力します。(これの計算に手元の環境で20分くらいかかりました、、マルチスレッド処理したいところですね、、)
```python
vectors_dynamic = {}

from tqdm.auto import tqdm
for idx in tqdm(vocab.values()):
    vectors_dynamic[idx] = model(input_ids=torch.tensor([[2,idx,3]])).last_hidden_state.squeeze(0)[1].detach().numpy().copy()
```
再計算はめんどくさいのでシリアライズしましょう
```python
import pickle

# 書き込み
with open('vectors_dynamic.pkl', 'wb') as f:
    pickle.dump(vectors_dynamic, f)

# 読み出し
with open('vectors_dynamic.pkl', 'rb') as f:
    vectors_dynamic = pickle.load(f) 
```

:::

# 単語のベクトル演算してみる

## 静的単語埋め込み
全章で準備をしたvectorsは単語idと単語の埋め込み表現をマッピングする辞書です。今回はWord2Vec等で有名な「王 - 男 + 女 = 女王」の計算をやってみたいと思います。
```python
import faiss
index = faiss.IndexFlatL2(len(vectors[0]))
index.add(token_embeds.detach().numpy())


vec = vectors[tokenizer.vocab['王']] - vectors[tokenizer.vocab['男']] + vectors[tokenizer.vocab['女']]

D, I = index.search(vec.reshape((1,768)), 10)
res = [tokenizer.ids_to_tokens[candidate] for candidate in I[0]]
print(res)
# ['王', '女', '女王', '国王', '王国', '王妃', '王家', '神', '##王', '島']
```
結果としては3番目に女王が出てきました。
「王+女」で試してみます。
```python
vec = vectors[tokenizer.vocab['王']] + vectors[tokenizer.vocab['女']]
D, I = index.search(vec.reshape((1,768)), 10)
res = [tokenizer.ids_to_tokens[candidate] for candidate in I[0]]
print(res)
# ['女', '王', '男', '女性', '女王', '娘', '女子', '神', '姫', '人']
```
王単体で調べてみます。
```python
vec = vectors[tokenizer.vocab['王']]
D, I = index.search(vec.reshape((1,768)), 10)
res = [tokenizer.ids_to_tokens[candidate] for candidate in I[0]]
print(res)
# ['王', '国王', '王国', '神', '皇帝', '女王', '##王', '王家', 'イギリス', '島']
```
この時点で女王という単語が割と上位に出てくるので、「王 + 子」のベクトルの近傍を調べたいと思います。王子が上位に上がってるのが理想ですかね。
```python
vec = vectors[tokenizer.vocab['王']] + vectors[tokenizer.vocab['子']]
D, I = index.search(vec.reshape((1,768)), 10)
res = [tokenizer.ids_to_tokens[candidate] for candidate in I[0]]
print(res)
# ['子', '王', '息子', '##子', '子供', '娘', '王子', '王国', '弟', '父']
```
王子が上位に上がってきました。なんとなく「子」の影響を強く受けているような気がします。ノルムを比較してみると
```python
print(f"王 = {np.linalg.norm(vectors[tokenizer.vocab['王']])}")
print(f"子 = {np.linalg.norm(vectors[tokenizer.vocab['子']])}")
print(f"女 = {np.linalg.norm(vectors[tokenizer.vocab['女']])}")
print(f"男 = {np.linalg.norm(vectors[tokenizer.vocab['男']])}")
# 王 = 1.0982342958450317
# 子 = 1.1643147468566895
# 女 = 1.134019136428833
# 男 = 1.125975251197815
```
これまでの結果と照らしあわすと、ノルムが大きい方のベクトルの影響を強く受けているような気がします。(ただしノルムを使って正規化しても結果は変わりませんでした。)

## 動的単語埋め込み
動的に生成される単語のベクトルを使って同じようにベクトル演算をしてみます。
まずは近傍探索ライブラリの準備をします。
```python
index = faiss.IndexFlatL2(len(vectors_dynamic[0]))
index.add(np.concatenate([[rep] for rep in vectors_dynamic.values()], axis=0))
```

```python
vec = vectors_dynamic[tokenizer.vocab['王']] + vectors_dynamic[tokenizer.vocab['女']] - vectors_dynamic[tokenizer.vocab['男']]
D, I = index.search(vec.reshape((1,768)), 10)
res = [tokenizer.ids_to_tokens[candidate] for candidate in I[0]]
print(res)
# ['王', '王位', '王宮', '王朝', '女', '宮', '宗', '王妃', '皇族', '后']
```

「王+子」で試してみると
```
['王', '子', '伯', '王妃', '宗', '余', '祖', '使', '##子', '王位']
```
「王＋女」で試してみると
```
['女', '王', '男', '顔', '王位', '王子', '后', '人民', '賊', '技師']
```
これをみると静的な単語ベクトルの方がベクトル同士の足し算の結果が直観に近く感じますね。

# 動的単語ベクトルの埋め込み空間を可視化してみる
全章の結果から、静的単語埋め込みと動的に獲得される単語ベクトルは少し勝手が違うように感じましたので可視化を試みます。
可視化にあたって手順は[こちら](https://zenn.dev/hpp/articles/d347bcb7ed0fc0)を参考にさせていただきました。静的な単語埋め込みの場合の可視化もこちらの記事から参照できます。

```python
import pickle

from transformers import AutoTokenizer
from sklearn.manifold import TSNE
import holoviews as hv
from holoviews import opts


tokenizer = AutoTokenizer.from_pretrained("cl-tohoku/bert-base-japanese-whole-word-masking")
vocab = tokenizer.get_vocab()
with open('vectors_dynamic.pkl', 'rb') as f:
    vectors_dynamic = pickle.load(f) 

tsne = TSNE(n_components=2, random_state=0)
reduced_vectors = tsne.fit_transform(list(vectors_dynamic.values())[:3000])

hv.extension('plotly')

points = hv.Points(reduced_vectors)
labels = hv.Labels({('x', 'y'): reduced_vectors, 'text': [token for token, _ in zip(vocab, reduced_vectors)]}, ['x', 'y'], 'text')

(points * labels).opts(
    opts.Labels(xoffset=0.05, yoffset=0.05, size=14, padding=0.2, width=1500, height=1000),
    opts.Points(color='black', marker='x', size=3),
)
```

結果はこんな感じになりました。パッとみた感じ上の方にサブワードのクラスタが、左下の方に数字のクラスタが存在することが見て取れます。
![](https://storage.googleapis.com/zenn-user-upload/e422ca65c9b1-20220706.png)

数字クラスタを拡大してみました。桁数が近いものが纏まっていますね、また隣り合う数字も近くに配置されているような気がします。
![](https://storage.googleapis.com/zenn-user-upload/abc130a16022-20220706.png)

国名、地域名のクラスタも存在しました。マスク言語モデルだけでも国名や地域名という外部知識は暗黙的に獲得できていそうです。
![](https://storage.googleapis.com/zenn-user-upload/7bfdabadbc87-20220706.png)

この辺りは漢字1字のクラスタです。
![](https://storage.googleapis.com/zenn-user-upload/b27391cd56ce-20220706.png)

この辺りをみると分かりやすいと思いますが、同じ文脈で登場しそうな単語が近い位置にあることがみて取れます。
![](https://storage.googleapis.com/zenn-user-upload/65d59931e074-20220706.png)

今回の結果からはどうして動的に獲得するベクトル表現だと「王+女-男=女王」といった演算が上手いこといかなかったかは分かりませんでした。
何か思うことがあった方はご連絡いただけると嬉しいです。

(追記)
よく考えたら、動的ベクトル表現を得る際に文脈として与えたのは[CLS]、[SEP]のみなのでAttention Layerでのデータの混ぜ合わせを行う際のノイズにしかならなさそうですね。。
本当はもう少しきちんとした文を入力したいですが比較条件を整えるのがなかなかに難しそうです。