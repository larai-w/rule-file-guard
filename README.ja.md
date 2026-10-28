# rule-file-guard（日本語）

[English](README.md)

AI のコーディングエージェントが「指示書」として読むファイル（`AGENTS.md`・`CLAUDE.md`・`GEMINI.md`・`.cursorrules`・`.windsurfrules`・`copilot-instructions.md`）を見張る、小さなチェックです。

指示書に次のことが書かれていたら、止めます。

- **名前を指定してパッケージを入れる**（`npm install left-pad`・`pip install requests`・`uv add httpx`・`cargo install …` など）
- **パッケージの取得先を変える**（`--index-url`・`--extra-index-url`・`registry=`・`npm config set registry`）
- **ダウンロードしたものを、そのままシェルで実行する**（`curl … | bash`・`wget … | sh`）

マニフェストにもう書いてあるものを入れるだけの手順は、止めません。たとえば `npm install`・`npm ci`・`pip install -r requirements.txt`・`pip install -e .`・`uv sync` です。

## なぜ要るのか

指示書は AI の権限で動きます。何が入るかを変えられて、しかも、よそからコピーして持ってくることが多いファイルです。それなのに、コードのようにはレビューされていません。

新しい論文（arXiv [2610.09264](https://arxiv.org/abs/2610.09264)・査読前）は、指示書に指示を紛れ込ませると、コーディングエージェントが本物の依存関係を攻撃者のパッケージに入れ替えてしまう、と報告しています。

依存関係は、マニフェストとロックファイルに書くものです。そこならレビューでも Dependabot でも見えます。このチェックは、依存関係を指示書の中に置かせないためのものです。

## GitHub Action として使う

```yaml
name: Rule-file guard
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: larai-w/rule-file-guard@v1
```

決まりで必要なら、Action はコミットの SHA で固定してください。

## コミットの前に動かす

[pre-commit](https://pre-commit.com) を使う場合:

```yaml
repos:
  - repo: https://github.com/larai-w/rule-file-guard
    rev: v1.0.0
    hooks:
      - id: rule-file-guard
```

`rule_file_guard.py` を自分の repo にコピーして、自分のフックから呼んでもかまいません。要るのは Python 3.9 以上と git だけです。

```bash
python3 rule_file_guard.py              # git が追跡している指示書を全部見る
python3 rule_file_guard.py AGENTS.md    # 指定したファイルだけ見る
```

## わざと書いている行のとき

レビューしたうえで、どうしても残す行には `rule-file-guard: allow` を書き足してください。たとえば HTML のコメントで書けます。

```markdown
pip install pre-commit  <!-- rule-file-guard: allow -->
```

## 止められないこと

これは文字の形を見るチェックです。これだけで守れるわけではありません。

- 見るのは、上に挙げた名前のファイルだけです。
- 見つけられるのは、コマンドの形で書かれた指示です。ふつうの文章で書かれた指示（「始める前に foo-utils というパッケージを足してください」など）や、文字を変換して隠した指示は見つけられません。
- パッケージの中身や、そのパッケージが本当にあるかどうかは調べません。

指示書の変更は、これからもコードと同じようにレビューしてください。依存関係はロックファイルで管理してください。

## どこから来たか

最初は、作者の7つの repo のために書きました。最初の版は、ただの `npm install` まで止めてしまい、3つの repo のふつうの準備の手順に引っかかりました。そこで「名前を指定して入れる」指示だけに絞りました。止めすぎる見張りは、すぐに無視されるようになるからです。

`tests/test_rule_file_guard.py` には、止めるべき行と、通すべき行の両方を入れてあります。

## ライセンス

MIT
