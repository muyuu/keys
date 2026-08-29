# keys

公開鍵のカタログと、そこから `authorized_keys` を組み立てる仕組み。

鍵の作成はここの仕事ではない。各マシンで作られた公開鍵を集め、どのホストがどの鍵を受け入れるかを
宣言して、`authorized_keys` に落とすところまでを扱う。

## 構成

```
keys/<name>.pub     公開鍵のカタログ。publish が書く
hosts/<label>       そのホストに入れる鍵の名前を 1 行ずつ。人間が書く
bin/publish         ~/.ssh の公開鍵をカタログへ登録する
bin/apply           カタログから authorized_keys を組み立てる
```

依存は git と POSIX sh だけ。受け入れ側のホストに用意するものは他に無い。

## 公開と認可を分けている

**`publish` は鍵をカタログに載せるだけで、どのホストにも入らない。** ホストに入れるには
`hosts/<label>` に名前を足して commit する。

公開鍵を配ること自体は無害なので自動化してよい。一方「その鍵でこのホストに入れる」は、実質的に
ログインできる人を増やす操作で、供給元に書ける人がそのままホストの shell を持つのと同じになる。
だから認可だけは commit として残し、履歴と差分で追えるようにしてある。

## 使い方

### 鍵を公開する（鍵のあるマシンで）

```sh
bin/publish                          # ~/.ssh/id_ed25519.pub を <ホスト名> として登録
bin/publish ~/.ssh/id_ed25519_work.pub   # <ホスト名>-work として登録
bin/publish -n laptop ~/.ssh/id_ed25519.pub   # 名前を明示
```

commit して push するところまで行う。`-P` で push を止められる。

### ホストに入れる（手元で）

`hosts/<label>` を作って、入れる鍵の名前を書いて commit する。

```
# hosts/vps1
wsl
laptop-work
```

ラベルは FQDN や IP ではなく不透明な名前にしておく。このリポジトリは public なので、
「どの鍵がどのホストに入るか」以上のことを晒す必要はない。

### 反映する（受け入れ側のホストで）

```sh
git pull
bin/apply -n        # dry-run。何が書かれるか確認する
bin/apply           # ~/.ssh/authorized_keys の管理ブロックを更新
```

ラベルは既定でホスト名。違うなら `bin/apply vps1` のように渡す。

## authorized_keys の扱い

`apply` はマーカーで囲った管理ブロックだけを書き換える。

```
# >>> keys: managed block >>>
...
# <<< keys: managed block <<<
```

ブロックの外にある行は読まないし消さない。手で足した鍵や、他の仕組みが入れた鍵はそのまま残る。
反映後に鍵が 1 本も残らない場合は書き込まずに中断する。

## 鍵を無効にする

カタログから `keys/<name>.pub` を消して commit し、各ホストで `bin/apply` を流す。

ここのカタログが代表するのは、ここで宣言したホストだけ。その鍵を他にも登録してあるなら
（サービスのアカウント等）、そちらは別途消す。
