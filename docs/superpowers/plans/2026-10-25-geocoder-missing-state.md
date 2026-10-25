# Geocoder の欠けた state の補完 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Nominatim の `address` に state が入らない住所（東京都）で、`display_name` から state を補い、Locations ダッシュボードの States パネルに Tokyo を出す。

**Architecture:** `TeslaMate.Locations.Geocoder.into_address/1` の state 行に、private 関数による補完を `||` でつなぐ。`reverse_lookup/3` と `details/2` はどちらも `into_address/1` を通るので、両方の経路に効く。既存の住所は、デプロイ後に本番で `refresh_addresses/1` を 1 回動かして埋める。

**Tech Stack:** Elixir、ExUnit、Mock（`Tesla.Adapter.Finch` のモック）、Nix devenv、GitHub Actions、GHCR、Ansible

**Spec:** `docs/superpowers/specs/2026-10-25-geocoder-missing-state-design.md`

## Global Constraints

- fork の中だけで直す。`gh` には常に `--repo rewse/teslamate` を付け、upstream に PR を出さない
- `elixir/lib/teslamate/locations/geocoder.ex` の upstream との差分は、state 行の変更と private 関数 1 つ（コメント付き）に留める
- 依存とロックファイル（`mix.lock`、`package-lock.json`、`Cargo.lock`）を変えない
- テストのフィクスチャには公開座標の応答か、本番データと無関係な架空の値だけを使う。本番の住所、座標、タイムスタンプ、本番ホストの接続情報を commit や PR に入れず、報告は件数だけにする
- コードコメントは英語で、現在のコードについて書く。commit メッセージは英語の Conventional Commits
- マージ、デプロイ（check と apply）、本番での `refresh_addresses/1` 実行は、それぞれ直前にユーザーの確認を取る

## Review Focus

- `display_name` や `address` がない応答（既存の "address fields" テストは `address` だけを返す）。state は nil のままで、例外を出さない
- 都と市の名前が同じ都市州（Berlin のように `ISO3166-2-lvl4` があり、州名が `city` と同じ値）。`display_name` の要素はすべて `address` の値と一致するので、state は nil のまま
- state がすでにあり、`display_name` に `address` にない要素もある住所。state は元の値のまま
- 先頭の POI 名など、`address` にない要素が複数ある `display_name`。最後の要素（都道府県レベル）を選ぶ
- `refresh_addresses/1` が使う `/lookup` の応答（リスト）。`details/2` でも state が補われる

---

### Task 1: `display_name` から state を補う

**Files:**

- Modify: `elixir/lib/teslamate/locations/geocoder.ex`（`into_address/1` の `state:` 行と、その下に private 関数を追加）
- Test: `elixir/test/teslamate/locations/geocoder_test.exs`（先頭に alias、末尾に `describe "missing state"` を追加）

**Interfaces:**

- Consumes: なし
- Produces: `defp state_from_display_name(raw :: map()) :: String.t() | nil`。`into_address/1` の中で `state: get_first(raw["address"], @state_labels) || state_from_display_name(raw)` として使う。公開 API（`reverse_lookup/3`、`details/2` の戻り値の形）は変えない

- [ ] **Step 1: 失敗するテストを書く**

`geocoder_test.exs` の `alias TeslaMate.Locations.Geocoder` の下に `alias TeslaMate.Locations.Address` を足し、モジュール末尾に次の describe を追加する。`@shinjuku` は公開座標（東京都庁付近、35.6896, 139.6917）の実際の応答から必要なキーだけを残したもの。ほかのフィクスチャは架空の値。

```elixir
  describe "missing state" do
    # Reverse lookup near the Tokyo Metropolitan Government Building on
    # nominatim.openstreetmap.org (Accept-Language: en). Tokyo appears only in
    # display_name because Nominatim labels it "city", which Shinjuku holds.
    @shinjuku %{
      "address" => %{
        "amenity" => "Tokyo Metropolitan Office of Postal Service",
        "city" => "Shinjuku",
        "country" => "Japan",
        "country_code" => "jp",
        "ISO3166-2-lvl4" => "JP-13",
        "neighbourhood" => "Nishi-Shinjuku 2",
        "postcode" => "160-0023",
        "quarter" => "Nishi-Shinjuku",
        "road" => "Tocho-dori Ave."
      },
      "display_name" =>
        "Tokyo Metropolitan Office of Postal Service, Tocho-dori Ave., Nishi-Shinjuku 2, Nishi-Shinjuku, Shinjuku, Tokyo, 160-0023, Japan",
      "osm_id" => 1_420_785_352,
      "osm_type" => "node"
    }

    setup_with_mocks([
      {Tesla.Adapter.Finch, [],
       call: fn %Tesla.Env{} = env, _opts ->
         {:ok, %Tesla.Env{env | status: 200, body: Process.get(:body)}}
       end}
    ]) do
      :ok
    end

    defp state_of(body) do
      Process.put(:body, body)
      {:ok, %{state: state}} = Geocoder.reverse_lookup(0.0, 0.0)
      state
    end

    test "fills the state from display_name inside a first-level subdivision" do
      assert state_of(@shinjuku) == "Tokyo"
    end

    test "fills the state from the last display_name part no address label holds" do
      body = Map.update!(@shinjuku, "display_name", &("Unlisted Place, " <> &1))
      assert state_of(body) == "Tokyo"
    end

    test "fills the state in details" do
      Process.put(:body, [@shinjuku])

      assert {:ok, [%{state: "Tokyo", city: "Shinjuku"}]} =
               Geocoder.details([%Address{osm_id: 1_420_785_352, osm_type: "node"}], "en")
    end

    test "keeps the state from the address labels" do
      body = %{
        "address" => %{
          "city" => "Osaka",
          "country" => "Japan",
          "ISO3166-2-lvl4" => "JP-27",
          "postcode" => "530-0001",
          "province" => "Osaka Prefecture"
        },
        "display_name" => "Unlisted Place, Osaka, Osaka Prefecture, 530-0001, Japan"
      }

      assert state_of(body) == "Osaka Prefecture"
    end

    test "keeps the state nil without a first-level subdivision code" do
      body = %{
        "address" => %{"city" => "Shinjuku", "country" => "Japan"},
        "display_name" => "Shinjuku, Tokyo, Japan"
      }

      assert state_of(body) == nil
    end

    test "keeps the state nil when every display_name part is an address value" do
      body = %{
        "address" => %{
          "city" => "Berlin",
          "country" => "Germany",
          "ISO3166-2-lvl4" => "DE-BE",
          "postcode" => "10178",
          "suburb" => "Mitte"
        },
        "display_name" => "Mitte, Berlin, 10178, Germany"
      }

      assert state_of(body) == nil
    end

    test "keeps the state nil without display_name or address" do
      assert state_of(Map.delete(@shinjuku, "display_name")) == nil
      assert state_of(Map.delete(@shinjuku, "address")) == nil
    end
  end
```

- [ ] **Step 2: テストが失敗することを確かめる**

Postgres が動いていなければ、リポジトリのルートで `direnv exec . devenv up --detached` を実行し、`direnv exec . sh -c 'pg_isready -h $DATABASE_HOST -p $DATABASE_PORT'` が成功するまで待つ。

Run: `direnv exec . sh -c 'cd elixir && mix test test/teslamate/locations/geocoder_test.exs'`
Expected: Tokyo を期待する 3 件（"fills the state from display_name ..."、"fills the state from the last ..."、"fills the state in details"）が `nil` で FAIL し、残りは PASS する

- [ ] **Step 3: `defp state_from_display_name(raw :: map()) :: String.t() | nil` を `geocoder.ex` に実装する**

`raw` が `"address"` に `"ISO3166-2-lvl4"` キーを持つマップで、`"display_name"` が文字列のときだけ、`display_name` を `", "` で区切り、`address` のどの値とも一致しない要素のうち最後のものを返す。それ以外は nil を返す（関数ヘッドのパターンマッチで分ける）。`into_address/1` の `state:` 行を `get_first(raw["address"], @state_labels) || state_from_display_name(raw)` に変える。関数の上には英語のコメントで次の2点を書く。Nominatim はラベルごとに 1 つの値しか残さないので、より詳しい要素とラベルが重なる第 1 階層の区分（place=city とリンクされた Tokyo）は `display_name` にしか現れない。`ISO3166-2-lvl4` の内側では、`address` にない最後の要素がその区分になる。

- [ ] **Step 4: テストが通ることを確かめる**

Run: `direnv exec . sh -c 'cd elixir && mix test test/teslamate/locations/geocoder_test.exs'`
Expected: 0 failures

- [ ] **Step 5: 全体のテストと lint を通す**

Run: `direnv exec . sh -c 'cd elixir && mix test'`
Expected: 0 failures

Run: `nix run .#lint`
Expected: 成功。ルートに旧レイアウトの `_build/` や `deps/` が残っていて mix-format が失敗する場合は、それを消してから再実行する

Run: `git -P diff upstream/main -- elixir/lib/teslamate/locations/geocoder.ex`
Expected: 既存の `@version` の差分に加えて、state 行と private 関数（コメント付き）の追加だけが出る

- [ ] **Step 6: Commit**

```bash
git add elixir/lib/teslamate/locations/geocoder.ex elixir/test/teslamate/locations/geocoder_test.exs
git commit -m "fix(geocoder): fill a missing state from display_name"
```

テストが終わったら `direnv exec . process-compose down` で devenv のサービスを止める。

### Task 2: マージ、デプロイ、既存住所の補完

**Files:** なし（GitHub、GHCR、本番ホストでの操作）

**Interfaces:**

- Consumes: Task 1 の commit を含むブランチ `fix/geocoder-missing-state`
- Produces: 本番の東京都の住所に state が入った状態

- [ ] **Step 1: push して PR を作る**

```bash
git push -u origin fix/geocoder-missing-state
gh pr create --repo rewse/teslamate --base main --head fix/geocoder-missing-state \
  --title "fix(geocoder): fill a missing state from display_name" --body-file <本文ファイル>
```

本文には、原因（東京都の境界が place=city とリンクされ、`address` から落ちること）、補完の条件、テスト内容、マージ後に `refresh_addresses/1` を実行する予定を書く。本番データは件数も含めて書かない。

- [ ] **Step 2: CI を待つ**

Run: `gh pr checks --repo rewse/teslamate --watch | cat`
Expected: すべて pass（`.github/` を変えていないので skip されない）

- [ ] **Step 3: ユーザーの確認を取ってマージする**

Run: `gh pr merge --repo rewse/teslamate --merge --delete-branch`

- [ ] **Step 4: GHCR イメージのビルドを待って確かめる**

main への push で "Build GHCR images" が自動で動く。`gh run list --repo rewse/teslamate --workflow ghcr_build.yml --limit 1 | cat` で run を見つけ、`gh run watch <id> --repo rewse/teslamate --exit-status` で待つ。

Run: `.agents/skills/teslamate-fork-upgrade/scripts/inspect_ghcr_images.sh main`
Expected: app と Grafana のイメージの revision がマージ後の main のコミットと一致する

- [ ] **Step 5: ユーザーの確認を取って check モードでデプロイを確かめる**

`~/git/ansible-playbooks` で:

Run: `direnv exec . ansible-playbook home-primary.yml --tags teslamate --check --diff`
Expected: teslamate のイメージ digest が新しい main に変わる差分だけが出る

- [ ] **Step 6: ユーザーの確認を取ってデプロイする**

Run: `direnv exec . ansible-playbook home-primary.yml --tags teslamate`
Expected: failed=0。`teslamate` コンテナが新しいイメージで起動している

- [ ] **Step 7: ユーザーの確認を取って既存の住所を取り直す**

実行前に、teslamate スキルの `query.sh` で `SELECT language FROM settings` を流して本番の言語設定を確かめる。ansible inventory の `home_primary` のホストに SSH し、`teslamate` コンテナで次を実行する（`docker` に権限がなければ `sudo` を付ける）。

Run: `docker exec teslamate bin/teslamate rpc 'TeslaMate.Locations.refresh_addresses("<language>")'`
Expected: エラーなく終わり、`docker logs teslamate` に例外がない

- [ ] **Step 8: 本番データで結果を確かめる**

teslamate スキルの `query.sh` で、`raw->'address'->>'ISO3166-2-lvl4'` ごとに state の値と件数を集計する。

Expected: `JP-13` の住所はすべて state が `Tokyo` で、NULL が 0 件。`JP-13` 以外の state ごとの件数は実行前と同じ。Locations ダッシュボードの States パネルの SQL（`$car_id` は 1、期間は過去 1 年）を流すと Tokyo が含まれる。ユーザーへの報告は件数だけにする。
