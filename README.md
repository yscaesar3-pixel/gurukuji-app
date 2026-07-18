# ぐるくじ ― Google Play 公開ガイド（広告収益化つき）

## このフォルダの中身
```
gurukuji-app/
├── index.html            … アプリ本体（PWA化・広告フック実装済み）
├── manifest.webmanifest  … PWAの設定ファイル
├── sw.js                 … オフライン対応（Service Worker）
├── icons/                 … アプリアイコン一式
├── PRIVACY_POLICY.md      … プライバシーポリシー下書き（広告掲載に必須）
└── README.md              … この手順書
```

まずはGoogle Play（Android）での公開を前提に手順をまとめています。iOSは後回しでOKです。

---

## ステップ0：広告の仕組みについて
`index.html` には、**Google AdMob** と連携するための下地（`AdManager`）をあらかじめ組み込んであります。

- ブラウザやPWAとして開いている間は何も起きません（`Capacitor` のネイティブ環境を検出した時だけ動作する設計です）
- バナー広告：アプリ起動時に画面下部へ表示する想定のスペースを確保済み
- インタースティシャル広告：リングが一定回数（目安6回）止まるごとに1回表示する呼び出しポイントを用意済み
- 実際に広告を出すには、後述の手順で **`@capacitor-community/admob`** プラグインを導入し、`index.html` 内のコメントアウトされている部分（`// const { AdMob }...` の行）を有効化し、ご自身のAdMob広告ユニットIDに書き換える必要があります

広告のタイミングや頻度（何回に1回出すか等）は `index.html` 内の `INTERSTITIAL_EVERY_N_STOPS` の数値を変更することで調整できます。頻度を上げすぎるとユーザー体験が悪化し、Google Playの「広告の頻度・配置ポリシー」違反にもなり得るため、最初は控えめな値から始めることをおすすめします。

---

## ステップ1：AdMobアカウントの準備
1. https://admob.google.com にアクセスし、Googleアカウントでサインアップ
2. 「アプリを追加」→ まだPlay Storeに公開していないので「いいえ」を選択し、アプリ名（例：ぐるくじ）を入力
3. 発行される **App ID**（`ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY` の形式）を控える
4. 広告ユニットを2つ作成
   - バナー広告ユニット（表示形式：バナー／アダプティブバナー）
   - インタースティシャル広告ユニット
5. それぞれの **広告ユニットID**（`ca-app-pub-XXXXXXXXXXXXXXXX/ZZZZZZZZZZ` の形式）を控える
6. EU/英国ユーザー向けの同意管理（UMP／User Messaging Platform）を、AdMobの「プライバシーとメッセージ」メニューから設定（GDPR対応として実質必須です）

---

## ステップ2：Capacitorでネイティブアプリ化
ご自身のPC（Node.js導入済み、Androidは Android Studio が必要）で作業してください。このチャット環境はネットワークアクセスがないため、実際のビルドはお手元の環境で行っていただく必要があります。

```bash
# 1. プロジェクト初期化
npm init -y
npm install @capacitor/core @capacitor/cli
npx cap init "ぐるくじ" "com.yourname.gurukuji" --web-dir=gurukuji-app

# 2. Androidプラットフォームを追加
npm install @capacitor/android
npx cap add android

# 3. AdMobプラグインを追加
npm install @capacitor-community/admob
npx cap sync android
```

### AndroidManifest.xml への追記
`android/app/src/main/AndroidManifest.xml` の `<application>` タグ内に、ステップ1で控えたApp IDを追加します。

```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY"/>
```

### index.html 側の広告IDを差し替え
`index.html` 内の `AdManager` 部分にある2箇所のコメントアウトを解除し、
```js
adId: 'ca-app-pub-XXXXXXXXXXXXXXXX/YYYYYYYYYY', // バナー
adId: 'ca-app-pub-XXXXXXXXXXXXXXXX/ZZZZZZZZZZ', // インタースティシャル
```
をステップ1で発行した実際の広告ユニットIDに置き換えてください。

### ビルド＆実機確認
```bash
npx cap open android
```
Android Studioが開いたら、実機またはエミュレータで動作確認します（最初は必ずAdMobの**テスト広告ID**で確認し、本番IDでのクリックはポリシー違反になるので避けてください）。

---

## ステップ3：プライバシーポリシーを公開する
広告を掲載するアプリは、プライバシーポリシーのURLが必須です。

1. `PRIVACY_POLICY.md` の内容を確認・編集（お問い合わせ先メールアドレスなどを記入）
2. どこかWeb上に公開する（GitHub Pages、Notionの公開ページ、Googleサイトなど無料の方法でOK）
3. 発行されたURLを控えておく（Play Consoleのストア掲載情報で入力します）

---

## ステップ4：Google Play Consoleでの公開作業
1. https://play.google.com/console でデベロッパー登録（**初回のみ$25の登録料**）
2. 「アプリを作成」→ アプリ名・言語・無料/有料（広告モデルなら「無料」）を設定
3. 左メニュー「アプリのコンテンツ」で以下をすべて入力
   - **プライバシーポリシー**：ステップ3のURL
   - **広告**：「はい、このアプリには広告が含まれます」を選択
   - **コンテンツのレーティング**：質問票に回答（アルコール等の質問には正直に「該当なし」で回答可能な内容になっています）
   - **対象年齢層**：一般的なツールアプリとして設定
   - **データセーフティ**：AdMob利用に伴い「広告ID」を収集・第三者共有する旨を申告（AdMobのドキュメントに申告例のテンプレートがあります）
4. 「ストアの掲載情報」で以下を用意
   - アプリ名・簡単な説明・詳しい説明
   - アイコン（512×512）
   - フィーチャーグラフィック（1024×500）
   - スクリーンショット（最低2枚、実機で撮影したもの）
5. 「本番」または、まずは**内部テスト**トラックでビルド（`.aab`ファイル）をアップロードして動作確認するのがおすすめです
   - Android Studioで `Build > Generate Signed Bundle` から `.aab` を作成
6. すべてのセクションが緑チェックになったら「本番環境」で公開審査に提出

審査は通常数時間〜数日です。差し戻された場合は、Play Consoleに理由が明記されるので、それに沿って修正して再提出してください。

---

## 今後お手伝いできること
- ストア用の説明文・スクリーンショット案の作成
- `capacitor.config.json` の中身の用意
- プライバシーポリシーの内容の詰め（お問い合わせ先の反映など）
- 広告表示ロジック（頻度・タイミング）の調整
