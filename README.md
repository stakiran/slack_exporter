# slack_exporter
Slack の指定チャンネル内の全メッセージを取得する Python スクリプト。

<!-- toc -->
- [slack_exporter](#slack_exporter)
  - [Overview](#overview)
  - [Requirement](#requirement)
  - [Usage](#usage)
  - [Sample](#sample)
    - [取得したいチャンネルの ID を得る](#取得したいチャンネルの-id-を得る)
    - [指定チャンネルのメッセージ一覧を得る](#指定チャンネルのメッセージ一覧を得る)
  - [FAQ](#faq)
    - [Q: 取得対象メッセージ総数と実際に取得した数が合ってないようなのですが？](#q-取得対象メッセージ総数と実際に取得した数が合ってないようなのですが)
    - [Q: IFTTT を使って Twitter のツイートを流してるチャンネルに対して実行すると本文が空になってますが？](#q-ifttt-を使って-twitter-のツイートを流してるチャンネルに対して実行すると本文が空になってますが)
  - [License](#license)
  - [Author](#author)

## Overview
指定チャンネル内の全メッセージを取得します。

前提:

- 私はひとりで Slack を使っています
- 私は [Legacy Token](https://api.slack.com/custom-integrations/legacy-tokens) を使用できます
- 指定チャンネル内の全メッセージをローカルにダンプしたいだけなんです
- とりあえず自分用に作ったので（特に標準出力内容とか）あまり整えてません

エクスポートの仕様:

- 取得するのはメッセージ本文のみ
  - :x: メッセージに付いてるリアクションや投稿者名
  - :x: アップロードしたファイル
  - ただしシステムメッセージも取得される＆そこにアップロード先URL等も書いてあるので **ポインタレベルなら漏れなく取得できてる** と思う
- **オレオレフォーマットの Markdown ファイルで出力します**

## Requirement
以下環境で動作確認しています。

- Windows7(x86 Professional), Windows10(x86_64 Home)
- Python2.7
  - requests

## Usage
- Slack にログインして Legacy Token を入手する
- 環境変数 `SLACKAPI_TOKEN` に Legacy Token をセットする
- `python slack_exporter.py -l` でチャンネル名とチャンネルIDの対応を得る
- `python slack_exporter.py -c (取得したいチャンネルのID)`

## Sample

### 取得したいチャンネルの ID を得る
チャンネルID は `CXXXXXXXX` の 9文字の文字列です。メッセージ取得にはこの ID が必要なので、まず最初にこのやり方で把握しておきます。

```
$ python slack_exporter.py -l
Asserting response of Get all channel list...
170626_tweet_slack(CXXXXXXXX):
170812_macos_custom(CXXXXXXXX):
170924_emptyroom(CXXXXXXXX):
general(CXXXXXXXX): This channel is for team-wide communication and announcements. All team members are in this channel.
hatena_hotentries(CXXXXXXXX):
tokyo_weathers(CXXXXXXXX):
links(CXXXXXXXX):
onlinesoft_mado(CXXXXXXXX):
onlinesoft_vector(CXXXXXXXX):
twitter_github(CXXXXXXXX):
...
```

### 指定チャンネルのメッセージ一覧を得る
`-c CXXXXXXXX` を指定して実行します。

```
$ python slack_exporter.py -c CXXXXXXXX
Asserting response of Get all channel list...
Asserting response of Get the number of history of channel "170626_tweet_slack"...
Message total 3652 items
Getting 1/4 from Latest to next 1000.
Asserting response of Get Channel History....
Getting 2/4 from 1491193787.769016 to next 1000.
Asserting response of Get Channel History....

$ dir log_170626_tweet_slack.md
2017/09/24  08:02           405,962 log_170626_tweet_slack.md
```

すると `log_(チャンネル名).md` というファイル名が作られます。ここにメッセージ一覧がオレオレフォーマットで書き込まれています。

以下はファイル内容のサンプルです。

```markdown
# 2017/08/12 14:07:05
<@UXXXXXXXX> has renamed the channel from "tweet" to "170626_tweet_slack"

ts:1502514425.163748, type:message

# 2017/06/26 09:33:21
つぶやきは slack から github に移行したのでもうここは使わないよー

ts:1498437201.045524, type:message

# 2017/06/26 09:11:56
今ふと思ったけど、TwDD ってローカルで HTML ファイルに吐かせさえすれば見やすさ的にも解決ではないかとふと思ったり。URL については投稿前に URL を正規表現で探して、そこを a タグでくるむとかすればいいわけだし。

ts:1498435916.943443, type:message

...(中略)...

# 2017/02/11 21:26:19
slot script から投稿 :smile:.

ts:1486815979.000008, type:message

# 2017/02/11 21:21:30
日本語ツイートテスト

ts:1486815690.000004, type:message

# 2017/02/11 21:21:23
Test tweet

ts:1486815683.000003, type:message
```

## FAQ

### Q: 取得対象メッセージ総数と実際に取得した数が合ってないようなのですが？
A: 原因不明です

まず「数が合っていない」例については以下をご覧ください。

```
$ slack_exporter.py -c (Channel-ID)
Asserting response of Get all channel list...
Asserting response of Get the number of history of channel "(Channel-Name)"...
Message total 3652 items   ★全部で3652件あると言っている
Getting 1/4 from Latest to next 1000.
Asserting response of Get Channel History....
Getting 2/4 from 1491193601.746383 to next 1000. ★実際取得したのは2000件以内(実際は1828件)
Asserting response of Get Channel History....
```

この例では 2000件 くらいの差が開いています。もし Message total が正しかったとすると 2000 件分をロストしていることになりますし、逆に 1828 件の方が正しかったとすると Message total は何やねん、という話になりますが……。

このような現象の原因はわかっていません。

一応技術的な手段の違いを述べておくと、

- Message total
  - [search.messages method | Slack](https://api.slack.com/methods/search.messages) で `in:(指定したチャンネルID)` をクエリにしてリクエストを行い、レスポンスの中に入ってる `response['messages']['paging']['total']` を表示している
- 実際の取得
  - [channels.history method | Slack](https://api.slack.com/methods/channels.history) で1リクエスト1000件取得、を繰り返すことで全件を取得している

という使用APIの違いはあります。（ちなみになぜ Message Total を使っているかというと、これを表示しないと「いつ終わるの？」と進捗がわからず精神衛生上よろしくないからです）

### Q: IFTTT を使って Twitter のツイートを流してるチャンネルに対して実行すると本文が空になってますが？
A: Slack の仕様だと思われます。

内部的には Slack API の [channels.history method](https://api.slack.com/methods/channels.history) の `text` フィールドの値を本文として保存してます。が、IFTTT で Twitter ツイートを取得するレシピを使って Slack に流し込んでいる場合、この text 値が空になるみたいです。

詳しいことはわかりません。IFTTT を使ったらアウトなのか。Twitter だからアウトなのか。はてさて……？

## License
[MIT License](LICENSE)

## Author
[stakiran](https://github.com/stakiran)
