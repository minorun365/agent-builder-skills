---
name: kb-demo-app-auth
description: デモアプリ・Webアプリの Cognito 認証のおすすめ構成。「Googleで続ける」か「メール（パスキーかメールのワンタイムコード。パスワードなし）」の2経路をCDKとフロントでどう組むか、パスキーが動かない設定の罠、Google client secret の置き場、ログイン画面のUX規約、Googleの同意画面に出るアプリ名の扱い。新しくデモアプリを作るとき、既存アプリの認証を作り直すとき、パスキーやGoogleログインが動かないときに読む。
user-invocable: true
---

# デモアプリの Cognito 認証（おすすめ構成）

デモアプリ・Webアプリの認証は、**特に指定がなければこの構成にする**：「Googleで続ける」か「メール（パスキーかメールのワンタイムコード）」。パスワードは使わない。

動く実物は [minorun365/marp-agent](https://github.com/minorun365/marp-agent)。
迷ったら実物を読む。設計の背景は同リポジトリの `docs/authentication-options.md`。

| ファイル | 中身 |
|---|---|
| `infra/lib/auth-stack.ts` | User Pool / App Client / Google IdP / Cognito ドメイン |
| `src/components/Auth/AuthScreen.tsx` | ログイン画面（326行。移植元） |
| `src/components/Auth/AuthScreen.css` | 同スタイル |
| `infra/lambda/auth/google-idp-manager/handler.ts` | Google IdP を作るカスタムリソース |
| `infra/lambda/auth/google-link/handler.ts` | preSignUp。既存メールユーザーへGoogleを連携 |
| `infra/lambda/auth/custom-message/handler.ts` | 確認コードとログインのコードのメールを日本語にする |

---

## 1. 利用者から見た形

| 経路 | 初回 | 2回目以降 |
|---|---|---|
| Google | 「Googleで続ける」で登録とログインが同時に終わる | Google |
| メール | メールアドレスだけで登録し、確認コードを入れるとそのままログイン | メールのワンタイムコード。パスキーを登録すれば顔・指紋 |

- **パスワードは使わない。** 画面に入力欄を置かず、App Client にもパスワードの経路を開けない（下の CDK）。
- **メールのコードは Cognito 標準メールで送る**（AWSアカウントあたり1日50通）。ログインのたびに1通なので、デモ規模なら収まる。上限に近づいたら SES の本番利用申請を検討する。
- **パスキーは任意。** 登録しなくても、メールのコードで期限なく使える。
- **Google 利用者へパスキー登録を案内しない。** 本人確認は Google 側が担当している。

## 2. ログイン画面のUX規約

1画面目は **2択だけ**に絞る。「Googleで続ける」と、メールアドレス入力＋「メールで続ける」。

メールを入れた次の画面では、**「パスキーでログイン」と「メールにコードを送る」を常に並べて出す**。存在しないアドレスでも Cognito は同じ応答を返すので、コードが来ない人への案内（登録し直しなど）はコード入力の画面に置く。

> **パスキー登録の有無で画面を出し分けない。** 出し分けると「そのメールアドレスが登録済みか」を第三者へ教えることになる。同じ理由で、存在しないアドレスも認証失敗も表示は「メールアドレスまたは認証情報を確認してください。」に統一し、Cognito 側も `preventUserExistenceErrors: true` にする。

メールのコードでログインした利用者には、**成功後に一度だけ**パスキー登録を案内する。

- 「パスキーを登録」と「あとで」の2つだけ
- 「あとで」を押されたら **30日間は再案内しない**（`localStorage` に押した時刻を持つ）
- 専用の設定画面は作らない。ログイン後のメイン画面も変えない

## 3. CDK（User Pool）

パスキーとメールのコードは、設定が1つでも欠けると**エラーにならず、ただ使えない**形で失敗する。次の5点をまとめて入れる。

```ts
this.userPool = new cognito.UserPool(this, 'UserPool', {
  featurePlan: cognito.FeaturePlan.ESSENTIALS,   // ① LITE ではパスキーを使えない
  mfa: cognito.Mfa.OFF,                          // ② 明示しないとOPTIONALへ補完されることがある
  selfSignUpEnabled: true,
  signInAliases: { email: true },
  signInCaseSensitive: false,
  autoVerify: { email: true },
  standardAttributes: { email: { required: true, mutable: true } },
  accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
  signInPolicy: {
    allowedFirstAuthFactors: { password: true, passkey: true, emailOtp: true },   // ③ password は Cognito の仕様で外せない（PasswordAuthenticationCannotDisabled）
  },
  passkeyRelyingPartyId: props.appDomain,                          // ④ 配信ドメインと完全一致させる
  passkeyUserVerification: cognito.PasskeyUserVerification.PREFERRED,
});

this.userPoolClient = this.userPool.addClient('WebClient', {
  generateSecret: false,
  preventUserExistenceErrors: true,
  authFlows: { user: true },    // ⑤ USER_AUTH だけ。パスワードの経路（userPassword・userSrp）を開けないことで入口を閉じる
  accessTokenValidity: cdk.Duration.minutes(60),
  idTokenValidity: cdk.Duration.minutes(60),
  refreshTokenValidity: cdk.Duration.days(30),
});
```

- **④ の RP ID は「利用者がアクセスするドメイン」**。`dxxxx.cloudfront.net` のような配信基盤の既定ドメインでもよいが、後からカスタムドメインへ移すと**登録済みのパスキーは全部使えなくなる**（RP ID が変わるため）。カスタムドメインを付ける予定があるなら、パスキーを入れる前に先に付ける。
- **自動の検査でログインが要るなら `adminUserPassword: true` だけ足し、`admin-initiate-auth` で入る**（AWS の認証情報が要るので、利用者向けの入口にはならない）
- **メールを日本語にする**：カスタムメッセージの Lambda トリガーで `CustomMessage_SignUp`（確認コード）と `CustomMessage_Authentication`（ログインのコード。パスワードなしのログインもここを通る）の件名と本文を差し替える
- `featurePlan: ESSENTIALS` は MAU 課金が発生する。デモ規模なら無視できるが、LITE からの変更なので認識しておく。

### フロント側（aws-amplify v6）

```ts
// パスキーでログイン
await signIn({ username: email, options: { authFlowType: 'USER_AUTH', preferredChallenge: 'WEB_AUTHN' } });

// メールのコードでログイン（nextStep は CONFIRM_SIGN_IN_WITH_EMAIL_CODE）
await signIn({ username: email, options: { authFlowType: 'USER_AUTH', preferredChallenge: 'EMAIL_OTP' } });
await confirmSignIn({ challengeResponse: code });

// パスワードなしで新規登録 → 確認コードでそのままログイン
await signUp({ username: email, options: { userAttributes: { email }, autoSignIn: { authFlowType: 'USER_AUTH' } } });
await confirmSignUp({ username: email, confirmationCode: code });   // nextStep は COMPLETE_AUTO_SIGN_IN
await autoSignIn();

// ログイン後にパスキーを登録
await associateWebAuthnCredential();
```

`Amplify.configure` の `Auth.Cognito` に `userPoolId` / `userPoolClientId` を渡すだけでよい。Google を使うときだけ `loginWith.oauth` を足す（次節）。

## 4. Google ログイン

### OAuth クライアントはデモごとに作らず、共通の1つを使い回す

**同意画面のアプリ名は GCP プロジェクト単位で1つ**なので、デモごとにクライアントを作っても名前は分けられない。
デモ用の中立な名前でプロジェクトとクライアントを1つ用意し、**以後のデモは全部これを使い回す**。

| 項目 | 決め方 |
|---|---|
| GCP プロジェクト | デモ専用に1つ作る |
| 同意画面のアプリ名 | 利用者に見えるのはこれ。特定のデモ名にせず、中立な名前にする |
| 公開ステータス | **本番環境**（テストユーザーの登録なしで誰でもログインできる） |
| Client ID | CDK のコンテキストで渡す（秘密ではない） |
| Client Secret | デプロイ先の AWS アカウントへ置く（下の「client secret の置き場」） |

**新しいデモを足すときの作業は2つだけ。**

1. Google Cloud Console でそのクライアントを開き、**承認済みリダイレクト URI に新しい Cognito ドメインの `/oauth2/idpresponse` を1行足す**
2. そのデモの AWS アカウントへ Secret を入れ、CDK に `googleClientId` を渡す

> **組織の Google Workspace 配下の GCP プロジェクトは、同意画面が「内部」（`orgInternalOnly`）になっていることがある。**
> その場合は組織のアカウントしかログインできず、外部の人が触るデモでは機能しない。プロジェクトを選ぶ前に同意画面のユーザータイプを確かめる。
>
> **`gcloud` でも API でも、クライアントの作成も編集もできない。リダイレクトURIの追記も含めて Console のブラウザ操作が唯一の手段**（2026-08 時点）。根拠は3つ:
> - `gcloud alpha iap oauth-clients` は IAP 専用で、ヘルプ自身が「プロジェクト内の全 OAuth クライアントの管理 API としては使えない」と明記。加えて IAP OAuth Admin API は 2026-03-19 に完全停止
> - `iap v1` の discovery を見ると `projects.brands.identityAwareProxyClients` の method は `create/get/list/delete/resetSecret` だけで、**更新系が無い**。停止していなくてもURIは足せない
> - Google の公開 API 一覧（`https://www.googleapis.com/discovery/v1/apis`）に、OAuth クライアントを管理する API 自体が存在しない
>
> **人に操作を頼むときは、クライアントの編集画面まで開けるURLを渡す。** URLはこの形:
> `https://console.cloud.google.com/auth/clients/<クライアントID>?project=<プロジェクトID>`

### 実装に必要なもの

**Cognito ドメイン**（Hosted UI 用）、**Google の OAuth クライアント ID / Secret**、**IdP 登録**、**App Client の OAuth 設定**。

```ts
supportedIdentityProviders: [cognito.UserPoolClientIdentityProvider.COGNITO, cognito.UserPoolClientIdentityProvider.GOOGLE],
oAuth: {
  flows: { authorizationCodeGrant: true },
  scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL, cognito.OAuthScope.PROFILE],
  callbackUrls: [`https://${appDomain}/`, 'http://localhost:5173/'],
  logoutUrls:   [`https://${appDomain}/`, 'http://localhost:5173/'],
},
```

フロントは `signInWithRedirect({ provider: 'Google' })` を呼ぶだけ。

**Amplify Gen2 の `defineAuth` では「スコープ」が2か所にあり、書き方が逆になる。** ここを取り違えると
デプロイも構成検査も通ったうえで、Googleのログイン画面が `invalid_scope` で開かない。

| 場所 | 何のスコープか | 書き方 |
|---|---|---|
| `externalProviders.scopes` | Cognito のアプリクライアントが出すトークンの範囲 | **大文字の列挙**（`['EMAIL','PROFILE','OPENID']`） |
| `externalProviders.google.scopes` | **Google へそのまま渡る生の文字列** | **小文字**（`['openid','email','profile']`） |

後者に大文字を書くと `authorize_scopes` が `"EMAIL PROFILE OPENID"` として IdP に入り、Google が
`invalid=[OPENID]` を返す（EMAIL / PROFILE は通ってしまうので、症状が OPENID だけに出て気づきにくい）。

**この種の失敗は AWS 側をいくら見ても分からない。** デプロイ後に Cognito の認可エンドポイントへ
リクエストを投げ、`accounts.google.com` のログイン画面へ着地するかを確かめる（サインインは不要）。
エラー時は `authError` クエリが base64 で理由を持っている：

```bash
curl -s -o /dev/null -L -w '%{url_effective}\n' \
  "https://<Cognitoドメイン>.auth.<region>.amazoncognito.com/oauth2/authorize?identity_provider=Google&client_id=<アプリクライアントID>&response_type=code&scope=openid+email+profile&redirect_uri=https%3A%2F%2F<配信ドメイン>%2F"
```

`.../signin/oauth/error?authError=...` へ着地したら失敗。`authError` を base64url デコードすると
`invalid_scope` / `redirect_uri_mismatch` などの理由が平文で読める。

### client secret の置き場

`cognito.UserPoolIdentityProviderGoogle` の L2 は **secret を CloudFormation テンプレートへ平文で残す**。marp-agent は、秘密値を state に残さないカスタムリソース（`google-idp-manager`）で IdP を作る形にしている。

> **組織の管理下にある AWS アカウントでは、SCP で `secretsmanager:CreateSecret` が拒否されていることがある。** marp-agent の handler は Secrets Manager から読む実装なので、その場合は **SSM Parameter Store の SecureString** に置き換える。Lambda 実行ロールに `ssm:GetParameter` と、SecureString を復号する `kms:Decrypt` を付ける。

### 段階的に入れてよい

Google は GCP 側の作業（OAuth クライアント作成、承認済みリダイレクトURIの追加）が要るので、**「クライアントIDが渡されたときだけ Google を有効にする」実装**にしておくと、パスキーだけ先にリリースできる。marp-agent は `this.node.tryGetContext('googleClientId')` の有無で分岐している。

### 同じメールアドレスの重複プロフィール

Cognito は、メールアドレスが同じでもローカルユーザーと Google ユーザーを**自動で統合しない**。放置すると同じ人に2つのプロフィールができる。marp-agent は preSignUp トリガー（`google-link`）で、確認済みメールアドレスが一致する既存プロフィールへ `AdminLinkProviderForUser` でリンクしている。デモアプリで既存利用者がいないなら、初期は省いてよい。

## 5. デモアプリでの判断（外部の人が触る場合）

- **Google の同意画面に出るアプリ名は GCP プロジェクトのブランディング**で決まる。クライアントIDを別に作っても、同じプロジェクトなら名前は同じ。**外部の人に見せるデモで、無関係な別アプリの名前が出る状態にしない。**
- 管理を楽にするなら、**デモ共通の中立な名前でプロジェクトとクライアントを1つ用意し、以後のデモはリダイレクトURIを1行足すだけ**にする。デモごとに作らない。
- セキュリティ要件の厳しい組織の利用者は、業務用 Google アカウントを外部アプリへ繋ぐことに抵抗がある。**Google を出しても、メール経路を同格で並べる**（片方だけにしない）。

## 6. 検証（実装完了と言う前に）

パスキーは「画面が出た」では動作確認にならない。次を実操作で通す。

1. メールで新規登録 → 確認コード → ログイン
2. パスキー登録 → いったんサインアウト → **パスキーでログイン**
3. パスキー未登録の状態でパスキーを押し、共通エラーが出てメールのコードへ戻れる。届いたコードのメールが日本語か
4. Google ログイン（有効にした場合）
5. **PC と iPhone の両方**（iOS Simulator か実機の Safari）

`localhost` は RP ID が `localhost` になるため、**本番ドメインで登録したパスキーはローカルでは使えない**。ローカルではメールのコードで確認する（画面だけを手元で起動し、本番の User Pool につないでよい）。
