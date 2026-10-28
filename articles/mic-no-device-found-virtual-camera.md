---
title: "ClaudeデスクトップやChrome内の通話でマイクが「No device found」になるのは仮想カメラが原因かもしれない"
emoji: "🎙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Windows", "Chrome", "Electron", "Claude" ]
published: true
---

## はじめに

Windows 版の Claude デスクトップアプリでボイスモードを使おうとしたところ、マイクが「No device found」と表示され、使えなかった。いろいろ試した結果、しばらく使っていなかった仮想カメラをアンインストールすると直った。検索しても同じ事例が見つからず、かなりはまったので記録しておく。

## 症状の確かめ方

マイクのレベルメーターは声に反応して動いている。Windows のプライバシー設定でもマイクは許可済みで、「現在使用中」とまで表示されている。しかし、アプリ側のデバイス一覧は空で、エラーも出ない。ブラウザで使う別の通話サービスでも同じことが起きていた。しかし、Discord や Firefox では普通に使えた。

マイク側の設定は次のようにかなりいじったが、どれでも直らなかった。

- Windows のマイク権限
- 既定デバイスの設定
- サンプリングレートを 48kHz に設定
- 排他モードをオフに設定
- 仮想マイクのアンインストール
- オーディオドライバーの入れ直し

今回の原因はカメラだった。Chrome で F12 を押してコンソールを開き、次を実行する。

```js
await navigator.mediaDevices.getUserMedia({video: true})
```

応答がなければ、仮想カメラが原因の可能性が高い。私の環境では `{audio: true}` はすぐに返るのに、`{video: true}` だけが返ってこなかった。また、Chrome を `--disable-features=MediaFoundationVideoCapture` 付きで起動すると、正常に応答が返るようになった。

## 直し方

使っていない仮想カメラ系のアプリをアンインストールする。私の環境では、古い NVIDIA Broadcast と古いバージョンの OBS の仮想カメラが原因だった。

アプリを消しても直らない場合は、カメラの登録だけが残っていることがある。PowerShell で次を実行すると、登録済みの仮想カメラの一覧が出る。怪しいものがないかを確かめられる。

```powershell
$cat = '{860BB310-5D01-11d0-BD3B-00A0C911CE86}'
'HKLM:\SOFTWARE\Classes\CLSID','HKLM:\SOFTWARE\WOW6432Node\Classes\CLSID' | % {
  Get-ChildItem "$_\$cat\Instance" -ErrorAction SilentlyContinue | % {
    [pscustomobject]@{Name=(Get-ItemProperty $_.PSPath).FriendlyName; Path=$_.PSChildName}
  }
} | Format-Table -AutoSize
```

## なぜカメラのせいでマイクが出てこないのか

Chromium 系(Chrome、Edge、Claude デスクトップのような Electron アプリ)は、マイクの一覧を取るときに `enumerateDevices()` でカメラも一緒に問い合わせているらしい。応答しない仮想カメラがあると処理が止まり、結果が返らない。そのため、マイクも一覧に出てこない。
