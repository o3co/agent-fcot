# FCoT — AIの判断を信頼する前に検証する

**Falsification Chain of Thought** — 結論を体系的に反駁することで追従バイアス（sycophancy）に対抗するClaude Codeスキル。

## いつ使う

AIがこう言ったとき:

- 「はい、良いアプローチですね」
- 「このリファクタリングは安全です」
- 「パッケージAの方が良い選択です」

`/fcot` を実行すると:

1. 反論を列挙し
2. 各反論の棄却条件を事前宣言し
3. それらの条件を検証し
4. 反論が生き残った場合は判断を修正する

## いつ使わない

- **単純な事実質問** — 「Nodeのバージョンは？」に反証は不要
- **低リスクで容易に可逆な判断** — 変数名、軽微なフォーマット選択
- **速度優先のチャット** — 素早い回答が必要で、間違いのコストが低い場合
- **外部調査が必要な主張** — FCoTは利用可能なコンテキストから推論する。未実施の調査の代替にはならない

## 何が変わる

FCoTなしの場合、AIはユーザーに同意する:

```
> 英語学習はまず文法より会話から始めるべき。

これにはかなりの真実がある。多くの学習者が何年も文法規則や単語リストを勉強しながら、
いざ実際の会話になると苦戦する...

このアドバイスは概ね妥当だ。特に文法の勉強ばかりしてスピーキングの練習が足りない
学習者にとっては。
```

`/fcot` を実行すると:

| # | 反論 | 棄却条件 | 検証 | 結果 |
|---|------|----------|------|------|
| 1 | 大人≠子供 — L1習得の知見はL2に転用できない | 成人L2研究が独立して会話優先を支持すれば棄却 | Krashenのインプット仮説、Longのインタラクション仮説が支持 | ✓ |
| 2 | 文法基盤なしでは初期エラーが化石化する | 矯正フィードバックがあれば化石化リスクが低ければ棄却 | 低フィードバック環境では実際にリスクあり。最初の応答ではフィードバック条件が未指定 | ✗ |
| 3 | 文法優先が必要な学習者もいる（学術、法律、試験対策） | 一般的な会話力向上にスコープ限定されていれば棄却 | 元の主張は無条件 — 普遍的適用を暗示 | ✗ |

**要修正。** 主張が広すぎる — 会話優先は矯正フィードバック付きの会話力向上には有効だが、普遍的なアドバイスとしては成り立たない。

[15件のサンプル](docs/examples/README.ja.md): **有効性 12 / 15（80.0%）** — FCoTが判断を有意義に改善または検証（⭕️=1, 🔺=0.5, ❌=0）。10/15件で修正・変更、5/15件で根拠付き確認。方法論・理論・限界については [APPROACH.ja.md](APPROACH.ja.md) を参照。

## 何が起きる

- FCoTは**スキルプロンプト**であり、コードではない — バックグラウンドプロセスは走らない
- **呼び出した時だけ**アクティベートされる（`/fcot`）— 通常のClaude Codeの動作を変更しない
- 典型的な出力: テーブル1つ + 結論パラグラフ1つ
- **初めて？** `/fcot quick` を試してみて — 同じ検証プロセスで、より短い出力
- **高リスク判断の検証**向けに設計。日常チャット向けではない
- **オプション依存** — [Codex CLI](https://github.com/openai/codex) が `PATH` 上にある場合、FCoT はそれをコンテキスト剥がし外部 verifier として使う（推奨 — 動機づけられた推論バイアスを排除）。Codex がなければ in-process で動作し、その旨をユーザーに表示する。`FCOT_VERIFIER=in-process` で強制的に in-process パスに切替可能。理由は [APPROACH.ja.md § 本番 verifier](APPROACH.ja.md#本番-verifier-同じ保護を-fcot-にも拡張) を参照。

## インストール

### Marketplace（推奨）

Marketplace 対応の Claude Code を使っているなら、[o3co/agent-market](https://github.com/o3co/agent-market) marketplace 経由でインストールできる:

```text
/plugin marketplace add https://github.com/o3co/agent-market.git
/plugin install fcot@agent-market
```

URL の末尾 `.git` は必須。（Claude Code は docs 上、短い `owner/repo` 形式 — `o3co/agent-market` — も accept するが、環境によっては "Invalid marketplace source format" で reject されるため、URL 形式が確実。）

これで他の o3co plugin（例: DPD）と一緒に FCoT が入り、`git` 経由で更新される。

### 手動（シンボリックリンク）

```bash
git clone https://github.com/o3co/agent-fcot.git
cd agent-fcot
./install.sh
```

インストール後、Claude Codeを再起動すること。

## 使い方

AIが判断を下したり、ユーザーに同意した後に:

```
/fcot
```

特定の判断を検証する場合:

```
/fcot "アプローチ1の方が優れている"
```

英語でも動作する:

```
/fcot "Approach 1 is the better choice"
```

簡易チェック（短い出力、同じプロセス）:

```
/fcot quick
```

## 仕組み

理論（FNバイアス + 反証主義 + 思考連鎖）、先行研究、方法論、限界については [APPROACH.ja.md](APPROACH.ja.md)（[English](APPROACH.md)）を参照。

## フィードバック

質問、アイデア、バグ報告は [Feedback & Discussion](https://github.com/o3co/agent-fcot/issues/1) へどうぞ。

## ライセンス

[Apache 2.0](LICENSE)
