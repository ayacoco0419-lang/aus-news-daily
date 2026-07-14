# 豪州金融サービスニュース収集タスク

---

## 目的
当タスクで洗い出した記事は、日本生命グループの豪州子会社に出向中の駐在員として、日々チェックしておくべき情報を効率的に把握するためのインプットとなる。

## 出力ルール（全セクション共通）
- 見出しは**元記事の英語表記のみ**（日本語訳を併記）
- 概要は**必ず日本語で要約**
- **ソース元のクリック可能なリンクを必ず記載**（リンクが確認できない記事は収録禁止）
- 承認ドメイン外の記事は収録禁止:
  - 規制当局: `apra.gov.au` / `asic.gov.au` / `rba.gov.au`
  - 経済金融: `bloomberg.com`
  - 豪州主要ニュース（Section B、媒体不問・以下のいずれか）: `abc.net.au` / `theage.com.au` / `news.com.au` / `theguardian.com` / `afr.com` / `theaustralian.com.au` / `9news.com.au` / `reuters.com` / `apnews.com` / `financialstandard.com.au` / `ifa.com.au` / `riskinfo.com.au` / `financialnewswire.com.au` / `insurancenews.com.au`
---

## 厳守ルール（最重要）

- **日付未確認の記事は出典を問わず一律で除外する。例外は一切認めない。**
  - 絶対日付・相対日付（"3 days ago" 等）のいずれかが明示されている場合のみ、SYDNEY_DATEから逆算して判定する
- クリック可能なURLが確認できない記事は収録しない
- COLLECTED_URLS（直近5回分の収録済みURL）に含まれるURLは除外する
- 同一トピックの記事が複数ある場合、最も重要な1件のみ全文要約し、2件目以降はタイトル（英語＋日本語）・媒体名・リンクのみ記載する
- **WebSearchツール呼び出し時に `allowed_domains`（ドメイン絞り込みオプション）を絶対に使用しない。** 複数ドメインを指定すると呼び出し自体が400エラーになることが確認済み。検索は必ずドメイン指定なしのプレーンクエリで実行し、返ってきた結果のURL文字列を後から目視で承認ドメインと照合して絞り込むこと（ツールのオプション機能ではなく手動フィルタで対応する）

---

## Step 1: 日付・期間設定

シェルコマンドを実行してSYDNEY_DATEを取得:


LOOKBACK_START = SYDNEY_DATE の2日前（48時間ウィンドウ）
（例: SYDNEY_DATE=2026-07-14 → LOOKBACK_START=2026-07-12）

---

## Step 2: 収録済みURLリストを取得（重複排除）

Notion MCPを使い、親ページID `354e36114e2381c5a8f0d5b8be95113f` の子ページ一覧を取得する。
最新5件のサブページ（タイトル「【豪州ニュース】...」）を取得し、各ページの本文に含まれるURLをすべて抽出して `COLLECTED_URLS` リストとして保存する。
Notion APIが失敗した場合は空リストで続行する。

---

## Step 3〜5: 記事収集フェーズ

**重要: Step 3〜5のすべての収集を完了させてからStep 6（日付検証・要約）に進むこと。**

---

## Step 3: Section A — 規制当局プレスリリース（APRA / ASIC / RBA）

| 規制当局 | 第1手段 | フォールバック |
|---|---|---|
| APRA | WebFetch: `https://www.apra.gov.au/news-and-publications` | 失敗時 → WebSearch `site:apra.gov.au news [年]` |
| RBA | WebFetch: `https://www.rba.gov.au/media-releases/` | 失敗時 → WebSearch `site:rba.gov.au media release [年]` |
| ASIC | WebSearch: `site:asic.gov.au media release [年]` | 結果が薄い場合 → WebSearch `ASIC media release Australia [年]` |

- LOOKBACK_START以降の新規リリースが1件もない場合は「該当なし」とし、直近の確認済みリリース（日付・タイトル）を参考情報として記載する

---

## Step 4: Section B — 豪州主要ニュース10本

**重要: このセクションのWebSearch呼び出しでは `allowed_domains` オプションを絶対に渡さないこと。** 複数ドメインを一度に指定すると400エラーになることが確認済み（単一ドメインを`site:`としてクエリ文字列に含める分には問題ない）。また、承認ドメインの多くはWebFetch・WebSearchのいずれからも直接アクセスできないケースが恒常的に発生している（"Claude Code is unable to fetch from ..."エラー等）。これはツール側の制約であり、クエリの工夫では解消しない。以下の優先順で取得する。

1. **RSSフィード直接取得（WebFetchで直接取得。第1手段）:**
   - ABC News: `https://www.abc.net.au/news/feed/51120/rss.xml`
   - The Guardian AU: `https://www.theguardian.com/australia-news/rss`
   - Reuters: `https://feeds.reuters.com/reuters/businessNews`
   - 9News: `https://www.9news.com.au/feed`
   - 取得できたRSSから `<item>` の `<title>` / `<link>` / `<pubDate>` を抽出する。個別のRSSが404等で失敗しても他のRSSは試し続けること。**多くの場合これらは全滅する（既知の問題）ため、失敗しても消耗せず速やかに手段2へ進むこと。**

2. **Bing News RSSフォールバック（第2手段・推奨。手段1で10本に満たない場合）:**
   以下のドメインごとに個別にWebFetchする（複数ドメインをORで結合すると精度が落ちるため、ドメインごとに分けて実行する）:

   `https://www.bing.com/news/search?q=site%3A{ドメイン}+Australia&qft=interval%3d%227%22&format=rss&mkt=en-AU`

   対象ドメイン: abc.net.au / theage.com.au / news.com.au / theguardian.com / afr.com / theaustralian.com.au / 9news.com.au / reuters.com / apnews.com / financialstandard.com.au / ifa.com.au

   - 取得できたRSSから `<item>` の `title` / `link` / `pubDate` を抽出する。
   - **`link`が`http://www.bing.com/news/apiclick.aspx?...&url=<URLエンコードされた実URL>`というリダイレクトラッパー形式になっていることがある（news.com.au・apnews.comで確認済みだが他ドメインでも起こりうる）。`link`が`bing.com`ドメインになっている場合は必ず`url=`パラメータをURLデコードして実際の記事URLを復元すること。**
   - あるドメインで0件返ってきても異常ではない（その日その媒体にAustralia関連の新着がないだけの可能性がある）。1ドメインの0件で全体を諦めず、残りのドメインを試し続けること。

3. **WebSearch（site:単一ドメイン指定。第3手段。riskinfo.com.au / financialnewswire.com.au / insurancenews.com.auの3ドメイン専用）:**
   この3ドメインはBing News RSSにインデックスされていないため手段2は使わず、WebSearchで直接取得する。**`allowed_domains`パラメータは使わず、クエリ文字列に`site:ドメイン名`を単一で含める形式（複数ドメイン同時指定は400エラーになるが単一なら問題ない）で実行する。**

   | # | クエリ |
   |---|---|
   | Q1 | `site:riskinfo.com.au [SYDNEY_DATEの年]` |
   | Q2 | `site:financialnewswire.com.au [SYDNEY_DATEの年月]` |
   | Q3 | `site:insurancenews.com.au [SYDNEY_DATEの年]` |

4. **WebSearch（プレーンテキスト。手段1〜3で10本に満たない場合のみ、最終フォールバック）:** 以下のクエリを**ドメイン指定なし（allowed_domainsパラメータなし）**のプレーンテキストで実行し、返ってきた結果のURLを承認ドメインリストと手動で照合して絞り込む

   | # | クエリ |
   |---|---|
   | Q1 | `Australia top news today [SYDNEY_DATEの月日]` |
   | Q2 | `Australia breaking news headlines [SYDNEY_DATEの月日]` |
   | Q3 | `Australia national news [SYDNEY_DATEの月日]` |

承認ドメイン（`abc.net.au` / `theage.com.au` / `news.com.au` / `theguardian.com` / `afr.com` / `theaustralian.com.au` / `9news.com.au` / `reuters.com` / `apnews.com` / `financialstandard.com.au` / `ifa.com.au` / `riskinfo.com.au` / `financialnewswire.com.au` / `insurancenews.com.au`）に該当するURLのみ候補として採用する。

---

## 10本の選定基準（候補が10本を超える場合の優先順位）

候補記事が10本を超える場合、以下の優先順位で上位10本を選定する（1が最優先。1〜3で10本に満たない場合のみ4から補充する）:

1. **豪州経済に関する記事**（RBA・金融政策・インフレ・雇用・CPI・GDP・経済見通し・IMF見通し・OECD見通しのいずれかに該当するもの）
2. **生命保険業界・APRA・ASICに関する記事**
3. **Acenda・Nippon Life・TALに関する記事**
4. **その他、豪州で注目度の高い政治・経済・社会ニュース**

10本に満たない場合は取得できた件数のみ収録し、「◯本のみ取得」と明記する。
---

## Step 5: Section C — ブルームバーグ 豪州経済・金融ニュース（最大8件）

**このセクションのWebSearch呼び出しでも `allowed_domains` オプションは使わない。** 以下のクエリをドメイン指定なしで実行し、結果から `bloomberg.com` のURLのみ手動で抽出する。

| # | クエリ |
|---|---|
| Q1 | `Australia economy RBA interest rate inflation Bloomberg [年]` |
| Q2 | `Australia superannuation insurance banking Bloomberg [年]` |
| Q3 | `Australian dollar AUD markets Bloomberg [年]` |

重要度順に上位最大8件まで収録。

---

## Step 6: 日付検証 + 内容取得

Step 3〜5で収集した全候補について:

1. WebFetchで記事本文と公開日を取得する
   - 公開日 >= LOOKBACK_START → 収録
   - 公開日 < LOOKBACK_START → **除外**
2. WebFetch失敗の場合はスニペットの日付を使う
   - 絶対日付または相対日付が明示されている → LOOKBACK_STARTと比較して判定
   - 日付が全く読み取れない → **例外なく除外**
3. WebFetch成功 → 本文から要約
4. WebFetch失敗だがスニペット日付が有効 → スニペットから要約し、末尾に「（スニペットより）」と明記

**重複排除:** COLLECTED_URLSに含まれるURLは除外する。

---

## Step 7: 要約作成フォーマット（記事1件ごと）


---

## Step 8: レポート作成

以下の構成でレポートをまとめる:

修正版のポイントは2つ。

1. **厳守ルールに `allowed_domains` 禁止を明記**（これが400エラーの直接原因）
2. **Step 4を「RSS優先→ダメならドメイン指定なしWebSearch＋手動フィルタ」に整理**（現行ファイルにはClaude Code側が自分でRSS案を追加してたので、それは活かしつつ順序を明確化）

```markdown
# 豪州金融サービスニュース収集タスク

---

## 目的
当タスクで洗い出した記事は、日本生命グループの豪州子会社に出向中の駐在員として、日々チェックしておくべき情報を効率的に把握するためのインプットとなる。

## 出力ルール（全セクション共通）
- 見出しは**元記事の英語表記のみ**（日本語訳を併記）
- 概要は**必ず日本語で要約**
- **ソース元のクリック可能なリンクを必ず記載**（リンクが確認できない記事は収録禁止）
- 承認ドメイン外の記事は収録禁止:
  - 規制当局: `apra.gov.au` / `asic.gov.au` / `rba.gov.au`
  - 経済金融: `bloomberg.com`
  - 豪州主要ニュース（Section B、媒体不問・以下のいずれか）: `abc.net.au` / `smh.com.au` / `theage.com.au` / `news.com.au` / `theguardian.com` / `afr.com` / `theaustralian.com.au` / `9news.com.au` / `skynews.com.au` / `reuters.com` / `apnews.com`

---

## 厳守ルール（最重要）

- **日付未確認の記事は出典を問わず一律で除外する。例外は一切認めない。**
  - 絶対日付・相対日付（"3 days ago" 等）のいずれかが明示されている場合のみ、SYDNEY_DATEから逆算して判定する
- クリック可能なURLが確認できない記事は収録しない
- COLLECTED_URLS（直近5回分の収録済みURL）に含まれるURLは除外する
- 同一トピックの記事が複数ある場合、最も重要な1件のみ全文要約し、2件目以降はタイトル（英語＋日本語）・媒体名・リンクのみ記載する
- **WebSearchツール呼び出し時に `allowed_domains`（ドメイン絞り込みオプション）を絶対に使用しない。** 複数ドメインを指定すると呼び出し自体が400エラーになることが確認済み。検索は必ずドメイン指定なしのプレーンクエリで実行し、返ってきた結果のURL文字列を後から目視で承認ドメインと照合して絞り込むこと（ツールのオプション機能ではなく手動フィルタで対応する）

---

## Step 1: 日付・期間設定

シェルコマンドを実行してSYDNEY_DATEを取得:
```
TZ='Australia/Sydney' date '+%Y-%m-%d'
```

LOOKBACK_START = SYDNEY_DATE の2日前（48時間ウィンドウ）
（例: SYDNEY_DATE=2026-07-14 → LOOKBACK_START=2026-07-12）

---

## Step 2: 収録済みURLリストを取得（重複排除）

Notion MCPを使い、親ページID `354e36114e2381c5a8f0d5b8be95113f` の子ページ一覧を取得する。
最新5件のサブページ（タイトル「【豪州ニュース】...」）を取得し、各ページの本文に含まれるURLをすべて抽出して `COLLECTED_URLS` リストとして保存する。
Notion APIが失敗した場合は空リストで続行する。

---

## Step 3〜5: 記事収集フェーズ

**重要: Step 3〜5のすべての収集を完了させてからStep 6（日付検証・要約）に進むこと。**

---

## Step 3: Section A — 規制当局プレスリリース（APRA / ASIC / RBA）

| 規制当局 | 第1手段 | フォールバック |
|---|---|---|
| APRA | WebFetch: `https://www.apra.gov.au/news-and-publications` | 失敗時 → WebSearch `site:apra.gov.au news [年]` |
| RBA | WebFetch: `https://www.rba.gov.au/media-releases/` | 失敗時 → WebSearch `site:rba.gov.au media release [年]` |
| ASIC | WebSearch: `site:asic.gov.au media release [年]` | 結果が薄い場合 → WebSearch `ASIC media release Australia [年]` |

- LOOKBACK_START以降の新規リリースが1件もない場合は「該当なし」とし、直近の確認済みリリース（日付・タイトル）を参考情報として記載する

---

## Step 4: Section B — 豪州主要ニュース10本

**重要: このセクションのWebSearch呼び出しでは `allowed_domains` オプションを絶対に渡さないこと。** 前回、複数ドメインを一度に指定したところ400エラーで全滅した実績がある。以下の優先順で取得する。

1. **RSSフィード（WebFetchで直接取得。第1手段）:**
   - ABC News: `https://www.abc.net.au/news/feed/51120/rss.xml`
   - The Guardian AU: `https://www.theguardian.com/australia-news/rss`
   - Reuters: `https://feeds.reuters.com/reuters/businessNews`
   - SMH: `https://www.smh.com.au/rss/feed.xml`
   - 9News: `https://www.9news.com.au/feed`
   - 取得できたRSSから `<item>` の `<title>` / `<link>` / `<pubDate>` を抽出する。個別のRSSが404等で失敗しても他のRSSは試し続けること（1つの失敗で全体を諦めない）

2. **WebSearch（RSSで10本に満たない場合のみ、フォールバック）:** 以下のクエリを**ドメイン指定なし（allowed_domainsパラメータなし）**のプレーンテキストで実行し、返ってきた結果のURLを承認ドメインリストと手動で照合して絞り込む

   | # | クエリ |
   |---|---|
   | Q1 | `Australia top news today [SYDNEY_DATEの月日]` |
   | Q2 | `Australia breaking news headlines [SYDNEY_DATEの月日]` |
   | Q3 | `Australia national news [SYDNEY_DATEの月日]` |

承認ドメイン（`abc.net.au` / `smh.com.au` / `theage.com.au` / `news.com.au` / `theguardian.com` / `afr.com` / `theaustralian.com.au` / `9news.com.au` / `skynews.com.au` / `reuters.com` / `apnews.com`）に該当するURLのみ候補として採用する。

10本に満たない場合は取得できた件数のみ収録し、「◯本のみ取得」と明記する。

---

## Step 5: Section C — ブルームバーグ 豪州経済・金融ニュース（最大8件）

**このセクションのWebSearch呼び出しでも `allowed_domains` オプションは使わない。** 以下のクエリをドメイン指定なしで実行し、結果から `bloomberg.com` のURLのみ手動で抽出する。

| # | クエリ |
|---|---|
| Q1 | `Australia economy RBA interest rate inflation Bloomberg [年]` |
| Q2 | `Australia superannuation insurance banking Bloomberg [年]` |
| Q3 | `Australian dollar AUD markets Bloomberg [年]` |

重要度順に上位最大8件まで収録。

---

## Step 6: 日付検証 + 内容取得

Step 3〜5で収集した全候補について:

1. WebFetchで記事本文と公開日を取得する
   - 公開日 >= LOOKBACK_START → 収録
   - 公開日 < LOOKBACK_START → **除外**
2. WebFetch失敗の場合はスニペットの日付を使う
   - 絶対日付または相対日付が明示されている → LOOKBACK_STARTと比較して判定
   - 日付が全く読み取れない → **例外なく除外**
3. WebFetch成功 → 本文から要約
4. WebFetch失敗だがスニペット日付が有効 → スニペットから要約し、末尾に「（スニペットより）」と明記

**重複排除:** COLLECTED_URLSに含まれるURLは除外する。

---

## Step 7: 要約作成フォーマット（記事1件ごと）

```
 [英語タイトル]
**（日本語タイトル）**
**日付:** YYYY年M月D日　**媒体:** [媒体名]
**URL:** [クリック可能なリンク]
**要約（本文/スニペットより）:**
[①何が起きたか ②背景 ③現状 ④今後の見通し ⑤日本生保への示唆 を含む日本語要約]
```

---

## Step 8: レポート作成

以下の構成でレポートをまとめる:

```
対象期間: [LOOKBACK_START] 〜 [SYDNEY_DATE]
作成日時: [SYDNEY_DATE]

 本日のKey Takeaways
（全セクションを横断した最重要ポイント3〜5本を箇条書き）

 Section A｜規制当局プレスリリース（APRA / ASIC / RBA）
 Section B｜豪州主要ニュース（承認ドメイン）
 Section C｜ブルームバーグ 豪州経済・金融ニュース

 収集統計（セクション別件数）
 403エラー・アクセス不可サイト一覧
 対象期間内の記事ゼロ確認済みソース
```

---

## Step 9: Notionページ作成

Notion MCP（`notion-create-pages`）を使用し、親ページID `354e36114e2381c5a8f0d5b8be95113f` の子ページとして作成する。

- **タイトル:** `【豪州ニュース】YYYY-MM-DD（合計X件）`
- **本文:** Step 8で作成したレポートをそのまま記載。URLはすべてクリック可能なリンクにすること。

---

## Step 10: 完了報告

以下を報告:
1. セクション別記事数（A/B/C・合計）
2. Notionページ作成の成否とURL
3. 403またはアクセス不可だったソース一覧
4. 対象記事0件だったソース一覧（あれば）
```

GitHub上での手順: [prompts/aus-news-daily.md](https://github.com/ayacoco0419-lang/aus-news-daily/blob/main/prompts/aus-news-daily.md) を開く → 右上の鉛筆アイコン（Edit this file）をクリック → 中身を全選択して上記に差し替え → 一番下の「Commit changes」でコミット。これでコミットまで済めば、次回実行時に反映されるはず。

---

## Step 9: Notionページ作成

Notion MCP（`notion-create-pages`）を使用し、親ページID `354e36114e2381c5a8f0d5b8be95113f` の子ページとして作成する。

- **タイトル:** `【豪州ニュース】YYYY-MM-DD（合計X件）`
- **本文:** Step 8で作成したレポートをそのまま記載。URLはすべてクリック可能なリンクにすること。

---

## Step 10: 完了報告

以下を報告:
1. セクション別記事数（A/B/C・合計）
2. Notionページ作成の成否とURL
3. 403またはアクセス不可だったソース一覧
4. 対象記事0件だったソース一覧（あれば）

