---
title: ".NET MAUI + AdMob で iOS 実機が起動時クラッシュ：原因は iOS デプロイメントターゲットが 15 未満だった"
emoji: "💥"
type: "tech"
topics: ["dotnet", "maui", "csharp", "admob", "ios"]
published: true
---

## 結論

.NET MAUI のアプリに `Plugin.AdMob` 3.0.2 で AdMob を組み込んだところ、**iOS 実機だけが起動と同時にクラッシュ**しました。最終的にクラッシュが消えたのは、私の環境のiOS のデプロイメントターゲットが14.2だったので 、これを15.0 に上げてからだった。

AdMob の iOS 側で使われる Google Mobile Ads のバインディング（`Jc.GMA.iOS`）が **iOS 15.0 以上を必須**としています。`csproj` の `SupportedOSPlatformVersion` と `Info.plist` の `MinimumOSVersion` を **どちらも 15.0 に上げる**のが、まず疑うべきポイントです。

「広告を入れたら iOS だけ落ちる」「`MobileAds.SharedInstance` が null になる」「Swift のシンボルが見つからない」あたりで検索してきた方は、まずデプロイメントターゲットを確認してみてください。

## 環境

- .NET 10
- .NET MAUI
- `Plugin.AdMob` 3.0.2（2026-02-01 リリース。執筆時点の最新）
- iOS 26 / 実機（iPhone）

## 症状：起動と同時にクラッシュ

Android では広告が表示されるのに、iOS の実機ビルドだけが起動直後に落ちます。シミュレータではなく実機で発生しました。

出力されていたのは、広告とは一見無関係に見える Swift ランタイムのシンボル解決エラーです。

```
INFO: Failed to look up symbolic reference at 0x1012d8157 - offset 89377 - symbol symbolic _____Sg ScP in /private/var/containers/Bundle/Application/5F22CC25-DF2F-46C4-89F4-ED73F4A5069C/CoinCalcApp.app/CoinCalcApp - pointer at 0x1012ede78 is likely a reference to a missing weak symbol
```

末尾の `a missing weak symbol`（参照先の弱シンボルが存在しない）が手がかりでした。「あるべきはずのシンボルがバイナリに無い」状態で、SDK が前提とする OS のバージョンを満たしていないときに起こり得ます。

## 先に試して効かなかったこと

原因がわからないうちは、思いつくものを片端から試しました。結果から言うと、次はどれも効きませんでした。

- .NET 9 へのダウングレード
- AdMob の初期化を、呼び出す場所やタイミングを変えて試す

## 原因にたどり着くまで

`Plugin.AdMob` の GitHub Issue を見にいくと、**Issue #56「Null Error with iOS Build on Physical Device」**（2025年10月）が見つかりました。

そこで報告されていた症状は、私のエラー文とは違っていました。報告者の環境（iPhone 16 Pro / iOS 26.0.1 / .NET 8）では、`AppDelegate.cs` の `CreateMauiApp()` で `NullReferenceException` が発生し、`MobileAds.SharedInstance` が null になる、というものです。エラーメッセージは私の Swift シンボルエラーと異なりますが、「iOS 実機で AdMob 初期化時に落ちる」という**根は同じ**でした。

このスレッドで解決策が示されていました。要点は、iOS のデプロイメントターゲットを 15.0 以上にすること。理由は、AdMob の iOS バインディングである [`Jc.GMA.iOS`](https://www.nuget.org/packages/Jc.GMA.iOS/) が **iOS 15.0 以上を要求している**から、というものでした。

## 自分のプロジェクトを確認したら 14.2 だった

そこで自分の設定を見ると、`csproj` も `Info.plist` も iOS **14.2** のままでした。SDK が求める 15.0 に届いていません。私の環境では、これが「弱シンボルが見つからない」という形で出ていた可能性が高いと考えています。

## 修正

`csproj` のデプロイメントターゲットを 15.0 に上げます。

```xml
<SupportedOSPlatformVersion>15.0</SupportedOSPlatformVersion>
```

あわせて `Info.plist` も 15.0 に。

```xml
<key>MinimumOSVersion</key>
<string>15.0</string>
```

両方を 15.0 にしてビルドし直すと、iOS のクラッシュは出なくなりました。

正直に書いておくと、ここに至るまでに上記のように複数の変更を重ねていたので、「15.0 への変更だけが決め手だった」と断言はできません。それでも、AdMob の iOS バインディングが iOS 15 以上を要求しているのは事実で、14.2 のままだったのは明確な不整合でした。同じ症状なら、まずここを疑う価値は高いと考えています。

## 同じ原因でも、エラーメッセージが違うこともある

今回いちばん共有したいのはここです。Issue #56 は `NullReferenceException`、私は Swift のシンボル解決エラーと、**表に出るエラーは別物**でした。それでも行き着く先は同じ「iOS デプロイメントターゲットが 15 未満」。エラーメッセージで検索すると別々の問題に見えてしまい、たどり着きにくいのです。広告を入れて iOS 実機だけ落ちるなら、エラー文が何であれ、まずデプロイメントターゲットを疑う価値があります。

---

このクラッシュは、個人開発アプリ「CoinCalcApp」を iOS / Android の両ストアに公開するまでに踏んだ地雷の一つです。土台にしていたフレームワークを手放した話、Android 側で起きた別のトラブル、そして審査を通すまでの全行程と、「AI とどう仕事を分担したか」は、書籍にまとめました。

『.NET MAUI 実戦記 〜詰んだ依存、ハマった広告、通した審査〜』
https://www.amazon.co.jp/dp/B0H4F5WKRZ

### 参考

- Plugin.AdMob Issue #56: Null Error with iOS Build on Physical Device — https://github.com/marius-bughiu/Plugin.AdMob/issues/56
- Jc.GMA.iOS（Google Mobile Ads iOS バインディング）: https://www.nuget.org/packages/Jc.GMA.iOS/
