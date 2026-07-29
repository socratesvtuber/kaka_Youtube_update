# kaka_Youtube_update
かかちゃんのYouTubeの予定をDiscordに配信するPG

## 概要
Google Apps Script (`Code.gs`) で動作する。登録済みYouTubeチャンネルのRSSフィード・YouTube Data APIを定期的に確認し、新規動画/配信の投稿や、配信状態（予定・配信中・アーカイブ）の変化をDiscordにWebhook通知する。

## スプレッドシート構成
- `channels` シート（列: A=チャンネル名, B=チャンネルID, C=チャンネルアイコンURL, D=Discord Webhookの識別子）
- `videoData` シート（列: A=タイトル, B=投稿日時, C=更新日時, D=動画ID, E=チャンネル名, F=配信状態(liveBroadcastContent), G=配信予定時刻, H=配信開始時刻, I=動画時間）

## 主な処理フロー
1. `fetchUpdateAndNotify()`（エントリーポイント。トリガーで定期実行想定）
   - `LockService` で多重実行を防止。
2. `loadAndVerifyChannelData()`
   - `channels` シートの各行を確認し、**アイコンURLがアクセス不可（`isUrlAccessible`がfalse）の場合のみ** `updateChannelIcon()` でYouTube APIから最新アイコンを取得し直す。
   - すなわち、通常運用ではアイコンURLが生きている限り再取得されない（=初回起動時など、URLが未設定/無効なときに取得される仕様）。
3. `updateAllChannels()` → `processChannelFeed()`
   - 各チャンネルのRSSフィード（直近5件）を取得し、`videoData` シートに未登録の動画があればYouTube APIで詳細情報を取得して新規投稿としてDiscordに通知・シートに追記。
   - 既存動画は `updateChecker()` で配信状態・配信予定時刻・タイトルの変化を検知し、変化があればDiscordに再通知。

## チャンネルアイコン取得の仕様
- `getChannelIcon(channelId)`: シートにアイコンURLが**未登録の場合のみ**YouTube APIから取得してシートに保存する（Discord投稿時のavatar_url取得に使用）。
- `updateChannelIcon(channelId)`: YouTube APIからチャンネル情報を取得し、該当行のアイコンURL列を強制的に上書きする。
- 上記の通り、既存の自動フローでは「アイコンURLが無効になった場合」または「未登録の場合」にのみ再取得される仕様であり、チャンネル側でアイコン画像自体が変更された場合（URL自体は生きたまま中身だけ変わるケースなど）は自動では更新されない。

### 手動実行用関数: `refreshAllChannelIcons()`
`channels` シートに登録された全チャンネルについて、URLの有効性を問わずYouTube APIから現在のアイコンを取得し、スプレッドシートの値を強制的に更新する。定期トリガーには含まれておらず、Apps Scriptエディタから手動で実行する想定。

## 更新履歴
- 2026-07-29: Codespace作成時にClaude Code CLIを自動インストールする `.devcontainer/devcontainer.json` を追加。
- 2026-07-29: `Code.gs` の仕様を解析し、本READMEに記載。あわせて、全チャンネルのYouTubeアイコンを手動で強制更新する関数 `refreshAllChannelIcons()` を追加。
