---
title: "tailscaleの設定"
---
## #️⃣tailscaleとは

https://tailscale.com/

Tailscaleは無料でVPN接続を管理してくれるサービスで、あらゆる場所にある端末を1つのネットワークに存在するかのように振舞わせることができます。

tailescaleをラズパイにインストール、外部からアクセスできるようにします。

https://tailscale.com/download/linux

DNSの指す先をラズパイのローカルアドレスからtailscaleが提供する100.64.0.0/10の範囲に変えてやることで、外出先でもvpnにさえつなげば接続できる状態になります。

MagicDNSを用いたCNAMEで指定できるのが理想だが、自分の環境ではうまくいかなかったためIPアドレスを直打ちしました。
