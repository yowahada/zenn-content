---
title: "余ったシンクライアント(Wyse 3040)を家庭内DNSサーバ(AdGuardHome)にした話"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["adguardhome", "dns", "ubuntu", "homelab", "自宅サーバ"]
published: false
---

**「軽量ディストリを選べば非力なマシンでも動くだろう」と思い込んで、インストールに2回失敗しました。**

余っていた Dell Wyse 3040（eMMC 8GB・メモリ2GBのシンクライアント）を、家庭内のDNSサーバにして、AdGuardHome で広告・トラッカーを家中まとめてブロックした話です。AdGuardHomeは、DNSの段階で、広告配信ドメインの問い合わせに `0.0.0.0` を返してブロックする仕組み（DNSシンクホール）です。端末ごとに広告ブロックのアプリを入れなくても、DNSの向き先を変えるだけでスマホにもPCにも効きます（ただし、うちのルーターでは家中への一括配布ができず、端末ごとに設定しました。理由は後半）。

失敗もハマりもそのまま書くので、同じくシンクライアントやeMMC 8GB級の小型機を再生したい人の役に立てば（結論だけ欲しい人は「まとめ」へ）。

:::message
**対象読者**：余ったx86小型機（シンクライアント等）がある／家中の広告を一括で消したい人。
**必要なもの**：常時起動できる小型機・USBメモリ（OS書き込み用）・家庭内LAN。Linuxの基本操作とSSHが分かると楽。
:::

## 何ができたか

- 家のPC・スマホのDNSをこのWyseに向けるだけで、広告・トラッカーのドメインが `0.0.0.0` に落ちてブロックされる
- 動作は軽い。導入直後でメモリ 77MB / ディスク 3.5GB使用・2.8GB空き。1か月あまり動かした今もメモリは約100MB
- 常時稼働させ、毎朝ヘルスレポートがLINEに届くようにして死活監視も兼ねた

![自宅LANの構成：端末のDNSはWyse(AdGuardHome)へ、通常の通信はGW経由](/images/home-lan-normal.png)
*端末のDNSだけをWyse(AdGuardHome)に向け、広告・トラッカーをブロックする。管理はMacからSSH。*

## なぜWyse / なぜAdGuardHome

- **Wyse 3040**：ファンレス・低消費電力で24時間つけっぱなしに向く。何より余っていた。ボトルネックは非力なCPUではなく**eMMC 8GBという狭いディスク**で、これが後段の失敗に直結する。
- **AdGuardHome**：単一バイナリで動き、自動更新も内蔵している。dnsmasq等を組み合わせるPi-holeに比べて動かすものが1つで済み、この機体に向いていた。Dockerではなくネイティブインストールを選んだ（理由は後述）。

## セットアップ手順

OSは紆余曲折あって Ubuntu Server 24.04.3（minimized） に落ち着いた（→ 失敗談は後半）。以降はSSHでMacから作業した前提。

### 1. タイムゾーン

```bash
sudo timedatectl set-timezone Asia/Tokyo
```

初期状態はUTC。クエリログの時刻が9時間ずれるので最初に直す。

### 2. AdGuardHome本体の取得

```bash
URL=$(curl -sL https://api.github.com/repos/AdguardTeam/AdGuardHome/releases/latest \
  | grep -o 'https://[^"]*AdGuardHome_linux_amd64.tar.gz' | head -1)
cd /tmp
curl -sL -o AdGuardHome.tar.gz "$URL"
tar xzf AdGuardHome.tar.gz
sudo mv /tmp/AdGuardHome /opt/AdGuardHome
```

公式が案内する `curl | sh` ではなく、GitHubリリースから直接取得して手で配置した。何が入るかを明示的にしたかったため。

### 3. ポート53の解放（最重要）

Ubuntuは `systemd-resolved` が `127.0.0.53:53` を握っているので、そのままではAdGuardHomeが53番を使えない。

```bash
sudo cp /etc/systemd/resolved.conf /etc/systemd/resolved.conf.bak
echo "DNSStubListener=no" | sudo tee -a /etc/systemd/resolved.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

:::message alert
`DNSStubListener=no` **だけ**だと、`systemd-resolved` が53番を手放す一方で `/etc/resolv.conf` は `nameserver 127.0.0.53` を指したままになり、**このマシン自身の名前解決が死ぬ**。参照先を `/run/systemd/resolve/resolv.conf`（上流DNSが直接書かれた実体）へ張り替えて初めて成立する。
:::

`:53` が空いていて、名前解決が生きていればOK。確認のコマンドは次のとおり。

```bash
sudo ss -tulnp | grep ':53 '
getent hosts github.com
```

### 4. サービス登録

```bash
cd /opt/AdGuardHome
sudo ./AdGuardHome -s install
```

`-s install` でsystemdユニット作成・自動起動の有効化・起動まで一度に済む。

### 5. スリープの無効化

DNSサーバが止まると家中の名前解決が止まる。4つのスリープ系ターゲットをまとめて `mask` して発動経路を塞ぐ。

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

`mask` すると `/dev/null` へのシンボリックリンクが張られ、どこから呼ばれても起動しなくなる（`static` のままだとACPIイベント等から呼ばれうる）。

:::message
電源ボタンは殺していない（`logind.conf` はデフォルト）。物理的に安全に止める手段は残しておきたいため。
:::

### 6. 初期設定ウィザード

`http://<WyseのIP>:3000` を開いて設定。

| 項目 | 設定値 |
|---|---|
| 管理用ウェブUI | 全インターフェイス / ポート 80 |
| DNSサーバ | 全インターフェイス / ポート 53 |
| 管理者アカウント | 任意 |

完了後、管理画面は `:3000` からポート番号なしのURLに変わる。

:::message alert
管理UIは家庭内LAN前提。ルーターのポート開放などで**WAN（インターネット）側に絶対に晒さない**。管理者パスワードは強固なものにする。
:::

### 7. 上流DNSをDoHにする

AdGuardHomeが自分で答えられない（＝ブロック対象でない通常の）ドメインは、上流DNSに転送する。ここを DoH（DNS over HTTPS）にすると、問い合わせ内容がISPからも見えなくなる。今回は Quad9（`https://dns10.quad9.net/dns-query`）を上流に設定した。

:::message
非力な機体では DoH/DoT の TLS 処理が地味に負荷になる。体感が重ければ平文DNSに戻す判断もある（プライバシーと速度のトレードオフ）。
:::

## ハマりどころ

ここが本題。軽量ディストリを選んで2回失敗した話から。

### 「軽量ディストリ」と「省ディスク」は別物だった

非力なマシンの再生といえば軽量ディストリ、という発想で最初に Lubuntu 24.04 を選んだ。これが誤り。Lubuntuがうたう「軽さ」はLXQtによるメモリ・CPU負荷の軽さであって、**ディスク使用量が小さいわけではない**。Wyse 3040のボトルネックは後者だった。

- **1回目（Normal Installation）**：インストール終盤で `外部のコマンドがエラーで停止しました` で中断。
- **2回目（Minimal Installation）**：追加DLも切って再挑戦したが同じ場所で落ち、今度はエラー全文が読めた。

```
コマンド update-initramfs が終了コード 1 で終了しました。
cpio: write error: No space left on device
E: mkinitramfs failure uncompressed cpio 2
```

`No space left on device`。パッケージ展開はできても、最後のinitrd生成に必要な一時領域が残っていない。デスクトップ環境を載せる前提では、削れるところを削っても入りきらない。

そもそもDNSサーバにGUIは1ミリも要らない、という当たり前に立ち返って Ubuntu Server (minimized) を入れ直したら何事もなく完走した。

| OS | 結果 | 備考 |
|---|---|---|
| Lubuntu 24.04 Normal | 失敗 | initrd生成時に容量枯渇 |
| Lubuntu 24.04 Minimal | 失敗 | デスクトップ環境だけでも入らない |
| Ubuntu Server 24.04.3 minimized | 成功 | 導入後 3.5G使用 / 2.8G空き |

インストール時は次のように選んだ。

- Installation Mode: Ubuntu Server (minimized)
- ストレージ: ディスク全体、LVMは無効化（数GBの単一用途機でLVMの利点が無く、全容量がrootに割り当てられない事態やinitramfsの肥大化を避ける）
- Install OpenSSH server にチェック（以降の作業を全部Macからやるため必須）

:::message
**教訓**：「軽量ディストリ」＝メモリ・CPUの話であって省ディスクとは限らない。eMMC 8GB級を再生するなら、デスクトップ版は最初から選択肢に入らない。用途がサーバなら素直にServer版。
:::

### インストールUSBを挿したまま再起動するとcloud-initがハングする

初回起動が `cloud-init ... running 'init-local'` で完全停止した。cloud-initが設定の供給元を探す際に、挿しっぱなしのインストーラUSBを候補と誤認して待ち続けるのが原因。USBを抜いて再起動したら一発で通った。

### Dockerではなくネイティブにした理由

当初はDocker + composeで組む予定だったが、実機スペックが判明した時点で変更した。

| | ディスク | メモリ |
|---|---|---|
| ネイティブ | 約40MB | 約50-100MB（実測77MB） |
| Docker | 約400-500MB + イメージ60MB | + dockerd常駐 100-200MB |

空き2.8GB・メモリ2GBの機体でDockerのオーバーヘッドは相対的に重い。AdGuardHomeは単一バイナリ＋自動更新内蔵なので、コンテナ化の利点（環境分離・更新容易）が薄いと判断した。

## 家庭内への展開と光BBユニットの壁

AdGuardHomeが動いても、端末が見に来なければ意味がない。本来はルーターのDHCPで「DNSは<WyseのIP>」と配布させたいが、**ソフトバンクの光BBユニットはこれができない**。

DHCP設定にあるのは「本機IP変更」「割当範囲」「固定割当（8個まで）」の3項目だけで、配布DNSサーバの入力欄が存在しない。探し方の問題ではなく、機能として無い。

取れる道は2つある。

| | 端末ごとに手動設定 | AdGuardHomeをDHCP化 |
|---|---|---|
| カバー範囲 | 設定した端末のみ | 家中すべて |
| DNS設定不可の機器(TV等) | 不可 | 可 |
| Wyse停止時の影響 | その端末のDNSのみ不通 | **家中がIP取得不可＝全断** |

前者（端末ごと手動）を採用した。光BB側のDHCPを止められない以上、後者だとDHCPサーバが2台共存して応答が不定になり、障害切り分けが困難になるため。

代わりにWyseのIPは光BBユニットの「固定割当」で固定した。

:::message alert
参照する端末はWyseのIPを直に指しているので、**IPが変わると設定済み端末が全滅する**。固定は必須。
:::

端末側は、iOSは「Wi-Fiの詳細 → DNSを構成 → 手動」でDNSだけ差し替え。AndroidはDNSだけを変えられず、IP設定を「静的」にする必要がある（「プライベートDNS」はDoT用でホスト名しか受け付けず、ローカルIPには使えない）。

:::message alert
セカンダリDNS（DNS 2）に `8.8.8.8` 等を入れてはいけない。AdGuardHomeがブロックしたドメインをそちらに聞き直して**ブロックが素通りする**ことがある。冗長性より一貫性を優先して空欄にした。
:::

## 常駐運用

DNSブロックのほかに、systemdタイマーで2つのジョブを回している（どちらも標準ライブラリのみ・外部依存ゼロ）。

- **日次ヘルスレポート（毎朝8時）**：ディスク空き・メモリ・CPU温度・eMMC消耗度・再起動検知をLINEに送る。正常時は数行、異常時だけ詳しく書く。レポートが届かないこと自体が異常のサイン＝死活監視を兼ねる。
- **株価モニタ（平日の夜）**：本題とは関係ないが、常時起動の機体があるついでに、監視している銘柄の値動きを調べてLINEに送っている。

ヘルスレポートのDNSの死活判定は `systemctl is-active` を信じない。プロセスが生きていても名前解決が死ぬことが実際にあったので、自前でDNSクエリを組み立てて `127.0.0.1:53` に投げ、回答が返るかで判定している。

eMMCの消耗度は次のコマンドで読める。

```bash
cat /sys/block/mmcblk0/device/life_time   # → 0x01 0x01 なら 0-10%
```

中古シンクライアント再利用では、この書き込み寿命が、地味だが見ておきたい指標になる。

## まとめ

- 余ったWyse 3040 + Ubuntu Server minimized + AdGuardHome で、家庭内DNSフィルタがメモリ77MBで常時稼働できた。
- 最大のハマりはOS選定。「軽量ディストリ ≠ 省ディスク」。eMMC 8GB級はServer版一択。
- 光BBユニットはDHCPで配布DNSを指定できない。DHCP2台共存の事故を避け、端末ごと手動設定を選んだ。
- 死活監視は「プロセス生存」ではなく「実際にDNSが引けるか」で見る。

次は、この分離のない家庭内LANで自分のスマホの通信を実際にMITMして「暗号化しても宛先は漏れる」を体感する記事を書く予定です（自分の端末・自分のネットワーク限定で、最後は対策で締めます）。

<!-- 続編（記事B: MITM）へのリンクを公開時にここへ -->
