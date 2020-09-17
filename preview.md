---
title: Application Gateway の散発的な 502 エラー
date: 2020-09-16 12:30:00
tags:
  - Network
  - ApplicationGateway
  - 502
  - Error
---
こんにちは、Azure サポート チーム檜山です。

今回は Application Gateway 経由で Web サイトへアクセスしている際にまれに 502 エラーが発生するという事象についてよくある事例を記載させていただきます。

---
まれに 502 エラーが発生するという事象についてよくある事例としては以下がございます。
<p> 	<li><a href="#backend-keepalive"><strong>バックエンドの Web サーバーにて HTTP KeepAlive が有効化されている</strong></a></li></p><p></p>
<p> 	<li><a href="#backend-issue"><strong>バックエンドが一時的に応答不可となり TCP セッションを切断している</strong></a></li></p><p></p>

以下のブログにも記載されておりますが、Application Gateway が HTTP Status 502 を返す理由はいくつかございますが、散発的に発生するような場合は本稿に記載した内容も一度ご確認いただけますと幸いです。

- [Application Gateway における 502 Error について](https://jpaztech.github.io/blog/archive/application-gateway-502-error-info/)



<h2 id="バックエンドの Web サーバーにて HTTP KeepAlive が有効化されている"><a href="#バックエンドの Web サーバーにて HTTP KeepAlive が有効化されている" class="headerlink" title="バックエンドの Web サーバーにて HTTP KeepAlive が有効化されている"></a><a href="#backend-keepalive">バックエンドの Web サーバーにて HTTP KeepAlive が有効化されている</a></h2>

HTTP KeepAlive は複数の HTTP のリクエストで TCP のセッションを使いまわし、通信を効率化するために利用されますが、Web サーバーにて HTTP KeepAlive が有効化されている場合に TCP の通信状況によってはまれに HTTP のリクエストが破棄されてしまうような事象が発生します。

以下のパケットは Chrome と Apache (HTTP KeepAlive : Timeout 5 秒) で動作を確認した際のパケットです。(<strong>※HTTP KeepAlive 動作確認のため、Application Gateway は利用しておりません</strong>)

```
10.0.1.4 が Web サーバー (Apache) 
167.220.223.186 がクライアント (Chrome)
```

No 97 で Web サーバーがレスポンスを返した 5 秒後にタイムアウトを迎え、FIN/ACK [No 99] を送りますが、そのタイミングでクライアントが HTTP リクエスト [No 100] を実施したため、そのリクエストについてはレスポンスが返らずに TCP セッションがクローズされていることを確認できます。

![Caputure](https://github.com/hisasima/appgw502/blob/master/capture.png)


上記のような クライアント - Web サーバー間のシナリオの場合、クライアント側に  502 エラーが記録されることはなく、ブラウザの動作によっては再接続で自動復旧する場合があります。

一方で、これと同様の事象が Application Gateway - バックエンドの Web サーバー間で発生した場合に Application Gateway は クライアントに 502 エラーを返す動作となります。(下図参照)

![AppGW502error](https://github.com/hisasima/appgw502/blob/master/AppGW502error.png)

そのため、散発的に 502 エラーが発生する場合、以下の点についてご確認いただけますでしょうか。
* HTTP KeepAlive が Web サーバーで有効化されていた場合、無効化し、事象が改善するか
* HTTP KeepAlive の無効化が難しい場合、KeepAlive のタイムアウトを数分程度に長くして事象が緩和されるか

一般的に HTTP KeepAlive によりエラーが発生する場合、クライアント側のタイムアウトより、サーバー側のタイムアウトを長くすることで対応可能となります。
 
<strong> Application Gateway V1 </strong> の場合、Application Gateway - バックエンド間で Application Gateway がクライアントとして動作する場合の HTTP KeepAlive のタイムアウトは固定で定まっておりません。
そのため、300 秒等の長い時間を設定いただき、502 エラーが緩和されるかをご確認いただくか、HTTP KeepAlive を無効化することをご検討ください。

<strong> Application Gateway V2 </strong> の場合、Application Gateway - バックエンド間で Application Gateway がクライアントとして動作する場合の HTTP KeepAlive のタイムアウトは 60 秒となることが確認できております。そのため、Web サーバー側で 90 秒や 120 秒といったタイムアウトを設定いただき、動作をご確認いただけますと幸いです。

<h2 id="バックエンドが一時的に応答不可となり TCP セッションを切断している"><a href="#バックエンドが一時的に応答不可となり TCP セッションを切断している" class="headerlink" title="バックエンドが一時的に応答不可となり TCP セッションを切断している"></a><a href="#backend-issue">バックエンドが一時的に応答不可となり TCP セッションを切断している</a></h2>

HTTP KeepAlive の問題ではない場合はバックエンドが一時的に応答不可となり TCP セッションを切断している可能性もございます。

再現の頻度によっては調査が難しくなる場合がございますが、高頻度で再現する場合や時間帯が決まっている場合は以下もご確認ください。

### バックエンドの Web サーバーにてパケットキャプチャを取得いただき、以下の点を確認

* Application Gateway からリクエストが届いているか
* Application Gateway のアクセスログと照らし合わせ、該当の 502 エラーのリクエストに対して、レスポンスを返しているか
	
### Application Gateway 非経由でアクセスし、以下を確認
* 遅延等なくレスポンスが返ってくるか
	
ただし、Application Gateway - バックエンド間が HTTPS であった場合は暗号化されているため、パケットの詳細が確認できず調査が難航する場合がございますため、バックエンドへのアクセスを一時的に HTTP に変更し、調査を行うこともご検討ください。


## FAQ
	
#### クライアント - Application Gatweay 間の Application Gateway 側のタイムアウトはいくつですか。
V1 SKU の場合、120 秒、V2 SKU の場合 75 秒です。

[キープアライブ タイムアウトと TCP アイドル タイムアウトの設定はどのようになっていますか?](https://docs.microsoft.com/ja-jp/azure/application-gateway/application-gateway-faq#what-are-the-settings-for-keep-alive-timeout-and-tcp-idle-timeout)

## 参考情報

Application Gateway のトラブルシューティングについては以下にも情報がございますので、ご参照ください。

[Application Gateway での無効なゲートウェイによるエラーのトラブルシューティング](https://docs.microsoft.com/ja-jp/azure/application-gateway/application-gateway-troubleshooting-502)

[Application Gateway のバックエンドの正常性に関する問題のトラブルシューティング](https://docs.microsoft.com/ja-jp/azure/application-gateway/application-gateway-backend-health-troubleshooting)
