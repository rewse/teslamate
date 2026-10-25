# Geocoder で欠けた state を display_name から補う

## 背景

Locations ダッシュボードの States パネルは `addresses.state IS NOT NULL` の住所だけを集計する。東京都内の住所は state がすべて NULL なので、Tokyo が出ない。

Nominatim は東京都の境界（R1543125、admin_level 4）を place=city の「Tokyo」ノードとリンクし、`linked_place=city`、`rank_address=16` として扱う。jsonv2 の `address` はラベルごとに最も詳しい値だけを残すため、`city` は区（東京駅なら Chiyoda）が取り、東京都は `address` から落ちて `display_name` にだけ残る。`geocoder.ex` は `state`、`province`、`territory` のキーから state を取るので NULL になる。本番にあるほかの県の住所には `province` が返っていて、影響を受けない。変更 #5785 の前の別名リストでも同じ結果になるため、退行ではない。

## 目的とスコープ

state が欠けた住所について `display_name` から都道府県レベルの名前を補い、States パネルに Tokyo を出す。fork の中だけで直し、upstream には出さない。

ゴール:

- `reverse_lookup` と、`refresh_addresses` が使う `details`（`/lookup`）の両方で補う
- 本番にある東京都の既存住所 60 件を埋める

非ゴール:

- upstream への PR
- ダッシュボード SQL の変更
- city など state 以外のフィールドの補完

アンチゴール:

- state がすでに取れている住所の値を変える
- Nominatim へのリクエストを増やす
- 都道府県レベルの区分がない場所に誤った state を入れる

## 制約

- upstream の `geocoder.ex` との差分を小さく保つ。fork のアップグレードのたびにコンフリクトの相手になるため
- テストのフィクスチャには公開座標（東京駅）の応答を使い、本番データを入れない

## 検討した方式

- A. `display_name` から補う。追加リクエストがなく、名前は Accept-Language に従い、`/lookup` の経路にもそのまま効き、差分も小さい。名前にカンマを含む要素があると取り違えることがある
- B. state が空のとき同じ座標を `zoom=5` で reverse し直す。Nominatim が `province` として返すので値は最も確かだが、東京の住所を作るたびにリクエストが倍になり、利用ポリシーのための待ち時間と、`/lookup` 経路での座標による追加問い合わせが要る。差分も最も大きい
- C. `ISO3166-2-lvl4` から名前への固定表を持つ。ネットワークに依存しないが、対応できる言語は表にある分だけで、他の県で同じことが起きても拾えない

A を採用した（ユーザーの選択）。リクエストも差分も増やさずに言語設定どおりの名前を使え、既存データも同じ経路で埋まるため。

## 挙動

`into_address/1` で、state が nil で、かつ `address` に `ISO3166-2-lvl4` があるときだけ補う。`display_name` を `", "` で区切り、`address` のどの値とも一致しない要素のうち最後のものを state にする。そのような要素がなければ nil のままにする（提案）。

`ISO3166-2-lvl4` は都道府県レベルの境界の内側にいることを示すので、これがない場所では補わない。最後の要素を選ぶのは、`display_name` が詳しい順に並び、都道府県レベルが国と郵便番号の手前に来るため（提案）。

本番の 60 件にこの規則を SQL で当てると、60 件とも Tokyo になった。

## 既存データの埋め方

デプロイ後、本番コンテナで `TeslaMate.Locations.refresh_addresses/1` を現在の言語設定のまま 1 回実行する。`/lookup` を 50 件ずつ呼ぶだけなので負荷は小さい。本番での実行になるため、その時点でユーザーの確認を取る（提案）。

## 実装者に任せること

- 補完関数の名前と置き場所（`geocoder.ex` 内の private 関数を想定）
- テストケースの構成。少なくとも、東京駅の応答で Tokyo が入る場合、state がすでにある場合、`ISO3166-2-lvl4` がない場合を含める
