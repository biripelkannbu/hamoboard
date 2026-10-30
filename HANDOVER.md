# 引継ぎメモ(biripel Connection)

このファイルは、**次の担当者(人でもAIでも)が最初に読むメモ**です。
アプリの作り方は `README.md`、データベースの設計図は `supabase/schema.sql` にあります。ここには、それらに書いていない「経緯・落とし穴・運用のルール」をまとめています。

> パスワード・招待コード・リカバリーコードなどの秘密は、このファイルに書かないこと(リポジトリはPublicです)。保管場所は「秘密の保管場所」を参照。

---

## 1. これは何か
サークル(ビリぺル)のバンド募集とメンバー紹介のための掲示板アプリ。名前は **biripel Connection**(リポジトリ名は `hamoboard` のまま)。

- フロント: 素のHTML/CSS/JS(ビルド不要)。GitHub Pagesで公開
- バックエンド: Supabase(認証・DB・自動処理)。無料プラン
- 費用: 0円。メンバー約100人の規模なら、無料枠で十分

## 2. 今の本番環境

| 項目 | 内容 |
|---|---|
| 公開URL | https://biripelkannbu.github.io/hamoboard/ |
| リポジトリ(本番) | https://github.com/biripelkannbu/hamoboard (`main`ブランチが公開される) |
| Supabase(本番) | プロジェクトID `ngsjjhaazqnlsuvcfahq`(サークル用GitHubアカウントの組織) |
| 管理アカウント | サークル用GitHub `biripelkannbu`。開発担当は共同編集者(Collaborator)として招待されている |
| keep-alive | `.github/workflows/keep-alive.yml` が3日おきにSupabaseへアクセス(Secretsの `SUPABASE_URL` / `SUPABASE_ANON_KEY` が必要) |

### 古い環境(使わない)
- 旧GitHub: https://github.com/Yotsuba7777/hamoboard (旧本番。**ここにはpushしない**。旧Supabaseにつながっており、サークル用とは別物)
- 旧Supabase: プロジェクトID `yrlpeuidthxccimukcux`
- テスト用Supabase: `vryjibnftznoplkdnoyw` (`hamoboard-test`)。不要になったら停止か削除

## 3. ファイルの役割

| ファイル | 役割 |
|---|---|
| `index.html` / `js/board.js` | 掲示板(募集の一覧・投稿・編集・締切・削除・♡リアクション) |
| `members.html` / `js/members.js` | メンバー紹介(名前・学年・パートで絞り込み、出席番号の表示) |
| `profile.html` / `js/profile.js` | マイページ(プロフィール編集、退会) |
| `signup.html` / `js/signup.js` | 新規登録(招待コードの確認、期・学年) |
| `login.html` ほか | ログイン、パスワード再設定 |
| `js/nav.js` | 上部メニューと通知ベル |
| `js/supabase-client.js` | Supabaseの接続先(URLと公開キー。公開してよい値) |
| `css/style.css` | 見た目 |
| `supabase/schema.sql` | DBの設計図(テーブル・権限・トリガー・自動削除) |

## 4. 仕様のポイント

- **招待コード**: `app_config` テーブルの `invite_code`。登録時に `check_invite_code()` で照合。値はSQLで変更(`update app_config set invite_code = '...' where id = 1;`)。コードの値はここに書かない
- **期と出席番号**: 期は数字のみ。出席番号は「期を3桁ゼロ埋め+その期の登録順を2桁ゼロ埋め」(例:17期の1人目=`01701`)。登録時、または期を初めて入力した時に、DBトリガー(`handle_new_user` / `handle_period_set`)が自動採番。**期と出席番号は、一度決まると本人は変更できない**(トリガーが元に戻す)。退会しても番号は再利用されない(`period_counters` は減らない)
- **学年**: 選択式(B1〜B4, M1〜M5, M6以上)。自動では上がらないので、毎年4月に各自が更新する運用
- **名前**: 「ニックネーム(LINEの名前)」の形を案内(強制はしていない)
- **ホスト**: `profiles.is_host = true` の人は、他人の募集も削除できる。編集・締切は投稿者のみ。付与はSQLで行う
- **募集**: 締切から30日たつと、pg_cronの `delete-old-closed-bands` ジョブが自動削除(元に戻せない)
- **通知**: 自分の募集にリアクションが来たらDBトリガーで `notifications` に1行作られ、ベルに表示(アプリ内のみ。メール・LINEには届かない)
- **リアクションの公開**: 募集ごとに、投稿者が「名前を公開するか」を選べる
- **退会**: マイページから `delete_own_account()` を呼び、`auth.users` を消す(関連データも連鎖して消える)
- **メンバーの削除(幹部が行う場合)**: アプリに画面はない。Supabaseの「Authentication → Users → Delete user」

## 5. 変更・デプロイの手順

1. コードを修正する
2. **DBの変更を伴う場合**: `supabase/schema.sql` を更新(本体の定義と、末尾の「機能追加分の反映」の両方)し、**先にSupabaseのSQL Editorで同じSQLを実行**してから、pushする(逆だと、反映までの間、保存エラーが出る)
3. ローカルのブランチ `circle-main` でコミットし、サークル用に push する:
   `git push circle circle-main:main`
4. 1〜2分でGitHub Pagesに反映。確認時はスーパーリロード(Ctrl+Shift+R)
5. **旧 `origin` には push しない**(旧本番が壊れる)

メモ: 確認用のテストは、テスト用Supabaseを作り、アプリのコピーの `js/supabase-client.js` だけを書き換えて行った。**テスト用のキーをリポジトリにコミットしないこと**。

## 6. 落とし穴(過去にハマったこと)

- **`schema.sql` と実際のDBがずれる**: 手作業でDBを変えた変更が、`schema.sql` に反映されていないことがあった(例:`reactions` テーブル、`delete_own_account` 関数。現在は追記済み)。DBを触ったら、必ず `schema.sql` にも反映する
- **出席番号の一括バックフィルが効かない**: 一括 `UPDATE` 自体が `profiles_handle_period_set` トリガーに引っかかり、番号が元に戻される。バックフィル中だけ `alter table profiles disable trigger profiles_handle_period_set;` → 実行 → `enable trigger` する
- **Supabaseのプロジェクト作成画面**: 「Automatically expose new tables」は**ONのまま**にする。OFFにすると、`schema.sql` にGRANTがないため、アプリが「permission denied」になる
- **RLSは絶対にOFFにしない**(SQL Editorの警告で「Run without RLS」を選ばない)。内部テーブル(`app_config`, `period_counters`)は、RLS有効・ポリシーなしで、誰も直接読み書きできない設計
- **Confirm email はOFF**(Authentication の設定)。ONだと登録後に確認メールが必要になり、アプリの流れが止まる
- **GitHub Actionsの定期実行**は、公開リポジトリで60日間動きがないと、自動で無効化されることがある。止まるとSupabaseが7日放置で一時停止する。Actionsタブでたまに確認し、止まっていたら再有効化する
- **Windowsの長いパス**: 作業用ファイルを、非常に深いフォルダに置くとパス長の上限(260文字)で失敗することがある。短いパスで作業する
- 秘密のキー(`sb_secret_…` / `service_role`)とDatabase passwordは、絶対にリポジトリ・チャットに載せない。アプリに使うのは `sb_publishable_…`(公開用)だけ

## 7. 不具合が出たときの流れ

1. 報告してもらう: 「どの画面で何をしたか」「出た表示(スクリーンショット)」「いつ頃」「全員か、その人だけか」
2. 切り分け(原因が多い順):
   - 全員が開けない → Supabaseが一時停止していないか(ダッシュボードの「Restore project」)、keep-alive
   - 「保存に失敗しました」 → 直近のコード変更に対応するSQLが、Supabaseに実行済みか
   - 反映されない → ブラウザのキャッシュ(スーパーリロード)
   - 特定の人だけ → その人のデータ(`profiles` など)をSQLで確認
3. 確認の場所: ブラウザのF12(Console / Network)、Supabaseの Logs、SQL Editorでの `select`
4. 直す → 上の「変更・デプロイの手順」へ。悪化したら `git revert` で戻す
5. 報告者に結果を連絡する

## 8. 運用のルール(幹部向けの要点)

- Supabase・GitHubを操作できる人は、**最低2人**にする
- 毎年3〜4月(新歓の前)に代替わり: 新会長・新開発担当を招待、管理者権限を移す、招待コードを変更、ホスト権限を付け替え、卒業生の整理、新入生への案内(学年の更新のお願いを含む)
- 学期の区切りに、Supabaseの Table Editor から `profiles` などをCSVでバックアップ

## 9. 秘密の保管場所(中身は書かない)

| 秘密 | 保管場所(担当者が記入) |
|---|---|
| サークル用GitHubのパスワード | |
| GitHubの二段階認証のリカバリーコード | |
| Supabaseの Database password | |
| 現在の招待コード | |

## 10. 未決・今後の課題

- GitHubの二段階認証の導入(未導入の場合)と、リカバリーコードの保管
- リポジトリ名・URL(`hamoboard`)を、アプリ名に合わせて変えるか
- 旧環境(旧GitHub・旧Supabase・テスト用Supabase)の片付け
- メンバーを削除する画面(現在はSupabaseの管理画面から)
- `README.md` を、今の仕様に合わせて更新する

## 11. AIアシスタントを使う人へ(このファイルを最初に読ませる場合)

- 日本語で、簡潔に答える
- **変更を加える前に、内容を確認する**(この開発では、そう頼まれている)
- アカウントの作成、パスワードの入力、秘密のキーの取り扱いは、AIには任せず、人が行う
- 作業はローカルの `circle-main` ブランチで行い、公開は `git push circle circle-main:main`(**`origin` には push しない**)
- DBの変更は、`schema.sql` とSQL Editor(人が実行)の両方が必要になる
