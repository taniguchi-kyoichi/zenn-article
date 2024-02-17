---
title: "【Go×GithubActions】タニタの体重計で計った体重を毎日指定時間にSlack通知する"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "GithubActions", "Slack", Docker, "HealthPlanetAPI"]
published: true
---

この記事では、タニタのHealth Planet APIを使用して体重情報を取得し、そのデータをSlackで毎日指定した時間に通知する方法をご紹介します。Go言語での実装からDockerコンテナ化、そしてGitHub Actionsを使ったスケジュール実行までを解説します。

完成形としては以下のような感じです。
![完成画像](/articles/images/health-slack.jpeg)

必要な工程は以下のようになります。
1. タニタのHealth Planet APIを新規登録する
2. Health Planet APIのアクセストークンをブラウザから入手する
3. タニタのAPIから身体情報を取得
4. データを整形してSlackWebhookを使って通知 
5. GoのコードをDockerコンテナ化してGithubActionsでスケジュール実行する

## タニタのHealth Planet APIの設定

まず、タニタの体重計を用意してください。私は以前にamazonでセール中に以下の体重計を購入しました

https://amzn.to/3T2jsOj

この体重計では、スマートフォンとBluetooth接続することができます。スマホのアプリ上でデータを登録するボタンを押すと自動的に体重計の電源が入って、計ったデータはBluetooth経由でhealth planetのデータベース上に登録されるため、非常に便利に使っています。


[ヘルスプラネット 健康管理アプリ](https://apps.apple.com/jp/app/id692700901)

APIの登録などはAPIのドキュメントなどを参考に進めてください。特に難しくはありません。

https://www.healthplanet.jp/apis/api.html

必要な工程としては、まずHealthPlanetに新規登録し、APIを有効にします。このとき、client_idとclient_secretが手に入るはずです。この情報を使って、ブラウザ上からOAuth 2.0認証を行います。この認証後、10分以内にアクセストークンを取得する必要があります。

## アクセストークンを取得

この部分は別にGoを使わなくてもcurlなどで実装すれば良いと思います。今回はWebアプリ化したくなったときのためにGoで実装してみました。

httpリクエストを投げてアクセストークンを入手しましょう。
```Go
import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
)

const (
	tokenURL = "https://www.healthplanet.jp/oauth/token"
)

// TokenResponse はアクセストークン取得のレスポンスを表します。
type TokenResponse struct {
	AccessToken string `json:"access_token"`
}

func GetAccessToken(clientID, clientSecret, code, redirectURI string) (string, error) {
	body := []byte(fmt.Sprintf("client_id=%s&client_secret=%s&redirect_uri=%s&code=%s&grant_type=authorization_code",
		clientID, clientSecret, redirectURI, code))
	req, err := http.NewRequest("POST", tokenURL, bytes.NewBuffer(body))
	if err != nil {
		return "", err
	}

	req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	respBody, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return "", err
	}
	fmt.Println("Response body:", string(respBody))

	var tokenResp TokenResponse
	if err := json.Unmarshal(respBody, &tokenResp); err != nil {
		return "", err
	}

	return tokenResp.AccessToken, nil
}
```

## タニタのAPIから身体情報を取得

次にタニタのhealthPlanetAPIから体重のデータを取得する実装をGoでやってみましょう。現時刻から7日前までの体重測定データをすべて取り出すような実装をしてみます。ちなみに、体重データは３ヶ月以内しか取得することはできません。

```Go
import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
)

const (
	inBodyInfoURL = "https://www.healthplanet.jp/status/innerscan.json"
)

// WeightDataResponse 体重データのレスポンスを格納するための構造体
type WeightDataResponse struct {
	Data []struct {
		Date  string `json:"date"`
		Value string `json:"keydata"`
		Model string `json:"model"`
		Tag   string `json:"tag"`
	} `json:"data"`
}

func GetWeightData(accessToken string, fromDate, toDate string) (*WeightDataResponse, error) {
	client := &http.Client{}
	req, err := http.NewRequest("GET", inBodyInfoURL, nil)
	if err != nil {
		return nil, err
	}

	// パラメータの設定
	q := req.URL.Query()
	q.Add("access_token", accessToken)
	q.Add("date", "1")
	q.Add("from", fromDate)
	q.Add("to", toDate)
	q.Add("tag", "6021") // 体重のデータタグ
	req.URL.RawQuery = q.Encode()

	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	respBody, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}

	fmt.Println(string(respBody))

	var weightData WeightDataResponse
	if err := json.Unmarshal(respBody, &weightData); err != nil {
		return nil, err
	}

	return &weightData, nil
}
```

単純にhttpリクエストを投げて、結果をマッピングしているだけです。重要な点としては、取得したいデータの種類によってtagが振り分けてあり、体重の場合は6021であるということです。
またfromやtoはyyyyMMddHHmmssの形式である必要があります。dateという項目はfrom, toの日付タイプを表しているらしいです。 0だと登録日付で1だと測定日付になります。任意の点のデータを登録することができるため、このような仕様になっているのではないでしょうか。基本的な要件では１にすれば良いと思います。

## データを整形してSlackWebhookを使って通知 

次に、このデータをslackに通知したいと思います。slack webhookのurl取得方法はたくさん情報が落ちていると思うので割愛します。

まず、main.goで、体重情報を取得したあと整形します。今回は、取得したデータの中で１番最近のデータを取り出します。

今日のデータが存在していたらそのデータを通知して、なければ「今日はまだデータがありません。毎日体重計に乗りましょう！」みたいな通知にしても面白いかなと思いました。また、先月の平均に対してどのくらい変化があったかを計算するロジックを差し込むのも良いかなと思いました。ひとまず、受け取ったデータに対して日付をパースしてslackに通知する関数に渡します。

```Go
now := time.Now()
fromDate := now.AddDate(0, 0, -7).Format("20060102150405") // 7日前
toDate := now.Format("20060102150405")                     // 現在の日付
fmt.Println(fromDate)
fmt.Println(toDate)

weightData, err := HealthAPI.GetWeightData(accessToken, fromDate, toDate)
if err != nil {
	fmt.Println("Error getting weight data:", err)
	return
}

if len(weightData.Data) > 0 {
	latestData := weightData.Data[len(weightData.Data)-1] // 最新のデータを取得
	date, err := time.Parse("200601021504", latestData.Date)
	if err != nil {
		fmt.Printf("日付のパースに失敗しました: %v\n", err)
		return
	}

	// Slackに送信するメッセージをフォーマット
	message := fmt.Sprintf("%sの体重：%skg", date.Format("1月2日"), latestData.Value)

	// Slackにメッセージを送信
	err = slack.SendMessageToSlack(webhookURL, message)
	if err != nil {
		fmt.Printf("Slackへの通知に失敗しました: %v\n", err)
		return
	}
} else {
	fmt.Println("体重データが見つかりません。")
}
```

slack通知の関数は以下のようになります。

```Go
import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
)

func SendMessageToSlack(webhookURL, message string) error {
	// Slackに送信するためのメッセージ構造体
	payload := map[string]string{"text": message}
	payloadBytes, err := json.Marshal(payload)
	if err != nil {
		return err
	}

	resp, err := http.Post(webhookURL, "application/json", bytes.NewBuffer(payloadBytes))
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("Slackにメッセージを送信できませんでした: %s", resp.Status)
	}
	return nil
}
```

非常に簡素な実装で済むのがとても良いですね！これで一通りの実装は終わりました。自分は彼女と２人のslackチームスペースを持っていて、その中で体重を自動投稿して共有するようにしています。私は自分の努力を人と共有することでやる気が上がるタイプなのだと思います。

最後にDockerコンテナ化してGithubActionsで定期実行できるようにしてみます。

## Dockerコンテナ化してGithubActionsでスケジュール実行する

まずは、ルートディレクトリにDockerfileを置きます。

```Dockerfile
# Goの公式イメージをベースにする
FROM golang:1.21.1-alpine

ENV TZ=Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# アプリケーションのソースコードを含むディレクトリをコンテナ内に作成
WORKDIR /app

# ホストマシンの現在のディレクトリ内のファイルをコンテナの作業ディレクトリにコピー
COPY . .

# 依存関係のインストール
RUN go mod download

# main.go を実行
CMD ["go", "run", "main.go"]
```
ここで必要なこととしては、タイムゾーンを日本にする必要があるということです。身体データを受け取る期間を日本時間で設定しないと時差が生まれてしまうというわけです。

最後に、.github/workflowsディレクトリに、dockerコンテナを実行するymlファイルを定義すれば完成です。
```yml
name: Run Docker Container

on:
  schedule:
    # UTCで13時 (日本時間で22時)
    - cron:  '0 13 * * *'
  workflow_dispatch:

jobs:
  build-and-run:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Build and Run Docker container
        run: |
          docker build . -t my-docker-app
          docker run my-docker-app
```
私は夜10時に定期実行することとしました。ここでもタイムゾーンを意識しないと時差が生まれてしまうようです。

ちなみにworkflow_dispatch:をつけるとgithub上から手動実行することが可能になります。この設定にしないとテスト実行してみるということができないため注意が必要です。

以上で実装は終わりです

## まとめ

この記事では、タニタの体重計で計った体重をGo言語で取得し、Slackで通知するプロセスを紹介しました。また、このプロセスをDockerコンテナにパッケージ化し、GitHub Actionsを使って定期的に実行する方法を解説しました。これにより、毎日の体重管理を自動化し、より効率的に行うことが可能になります。