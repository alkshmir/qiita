---
title: InteractionとWebhookでサーバレースなDiscord Bot作成
tags:
  - Go
  - discord
  - Bot
  - CloudRun
private: true
updated_at: '2024-07-29T20:59:22+09:00'
id: ed654f048b35f9748b0f
organization_url_name: null
slide: false
ignorePublish: false
---
# InteractionとWebhookでサーバーレスなDiscord Bot作成
[Interaction](https://discord.com/developers/docs/interactions/overview)ベースのDiscord Botを作ってみていい感じだったのでメモします。

## Discord Bot の2種類の作り方
Discord Botはユーザからのイベントを受け取って何らかの動作を行いますが、そのイベントの受け取り方は2種類あります：
- WebSocket
  - 人間のユーザと同じようにクライアント -> Discord へ接続します
  - Botプロセスを常駐させる必要があります
- Webhook
  - イベントが起こると Discord からあらかじめ定めたHTTPエンドポイント(Interaction Endpoint URL)へWebhookが飛びます
  - リクエストベースで起動する Lambda や Cloud Run などのサービスと相性が良いです

WebhookはWebSocketベースのBotよりも受信できるイベントが少なくなりますが、圧倒的にスケーラビリティとコスト効率に優れます。
従来のWebSocketベースのBotをクラウド環境にデプロイする場合、前述のようにBotが何もしてなくてもBotプロセスを常駐させる必要があるため、その間常に課金されることになります。
一方、Webhookを使う場合はリクエストが来たらその時だけ動的にリソースを消費するようなサーバーレスサービスを利用できます。ほとんどの趣味Botの場合、各クラウドサービスの無料枠に収まる使い方が可能なはずです。

また、書き味もAPIサーバーを書くのに似ているため、Botフレームワーク特有の挙動で悩むことがほぼないのも良かったです。

インタラクションイベントしか受信できない制限がユースケースにマッチすれば、Webhookベースで書く方が良いと思います。

## Goで実装
今回は[amatsagu/tempest](https://github.com/amatsagu/tempest)を使ってみました。
クライアントライブラリの一覧は[公式ドキュメント](https://discord.com/developers/docs/topics/community-resources#interactions)にまとまっていました。

https://github.com/amatsagu/tempest

example ディレクトリの実装例は執筆時時点でv1.2.0で動かなかったのでmainブランチを使いました。

```
go get github.com/amatsagu/tempest@fc7b5326c164f1c9c5f0c2c33446b7fabdd2783
```

全体の実装はこちらで公開しています。
https://github.com/alkshmir/random-song-bot

構成はとてもシンプルです。
```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	_ "net/http/pprof"
	"os"
	"strconv"

	"github.com/amatsagu/tempest"
	"github.com/joho/godotenv"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		slog.Info("Error loading .env file, using env variable")
	}
	port, err := strconv.Atoi(os.Getenv("PORT"))
	if err != nil {
		slog.Info("PORT environment variable not set, falling back to default 8080")
		port = 8080
	}

	slog.Info("Creating new Tempest client...")
	client := tempest.NewClient(tempest.ClientOptions{
		PublicKey: os.Getenv("DISCORD_PUBLIC_KEY"),
		Rest:      tempest.NewRestClient(os.Getenv("DISCORD_BOT_TOKEN")),
	})

	client.RegisterCommand(GetRandomSong)
	err = client.SyncCommands([]tempest.Snowflake{}, nil, false)
	if err != nil {
		slog.Error("failed to sync local commands storage with Discord API", err)
	}
	http.HandleFunc("POST /discord/callback", client.HandleDiscordRequest)

	slog.Info(fmt.Sprintf("Serving application at: :%d/discord/callback", port))
	if err := http.ListenAndServe(fmt.Sprintf(":%d", port), nil); err != nil {
		slog.Error("Fatal: ", err)
	}

}

var GetRandomSong tempest.Command = tempest.Command{
	Name:          "random-song",
	Description:   "send random song",
	AvailableInDM: true,
	SlashCommandHandler: func(itx *tempest.CommandInteraction) {
		songId, err := getRandomSongId()
		if err != nil {
			slog.Error("Error in getting random song")
			itx.SendLinearReply("error getting random song", true)
			return
		}
		post := generatePost(songId)
		itx.SendLinearReply(post, false)
	},
}
```

ボイラープレートを除くと、
- コマンドオブジェクトを定義して
- コマンドを登録・同期して
- `http.listenAndServe()`でリクエストを待つ

だけです。簡単ですね。
コード内で気をつけた方が良さそうなのは、`client.SyncCommands()`はギルドIDを指定しない場合グローバルにコマンドが登録される挙動になっているようですが、特にドキュメントされていない気がしました。


## デプロイ
ローカル環境で試す場合も、インターネットから接続可能なエンドポイントを用意する必要があるので、[ngrok](https://ngrok.com/)を使うのが簡便です。

```
ngrok http http://localhost:<port>
```

次にBotの設定画面で`INTERACTIONS ENDPOINT URL`にngrok CLIで表示された公開エンドポイントを貼り付けます。
上のコードベースではルートではなくて`<root>/discord/callback`で待ち受けているのでその辺はいい感じに設定します。

Save Changesボタンを押下するとDiscordからリクエストが飛んでエンドポイントへのReachabilityを確認してくれます。
設定がうまくいっていない場合は設定を保存することができません。

## Cloud Runにデプロイ
今回はCloud Runにデプロイしてみました。
特に難しいところはなく、環境変数とシークレットをポチポチ入れていくとデプロイできます。
環境特有なのは`PORT`環境変数で待受ポートを指定する点ぐらいと思われます。

デプロイが成功したらURLをコピーして、上述のINTERACTIONS ENDPOINT URLを更新します。

ほとんどコールドスタートだと思いますが、レイテンシは自分の用途では1~1.5sほどでほぼ気になりませんでした。

## まとめ
ユーザーの投稿を読んだりボイスチャンネルに繋いだりしないならInteractionベースで作った方が安上がりだし、書き味もシンプルなのでおすすめ

