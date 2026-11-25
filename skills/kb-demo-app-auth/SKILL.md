---
name: kb-demo-app-auth
description: デモアプリ・Webアプリの Cognito 認証のおすすめ構成。「Googleで続ける」か「メール（パスキーかメールのワンタイムコード。パスワードなし）」の2経路をCDKとフロントでどう組むか、パスキーが動かない設定の罠、Google client secret の置き場、ログイン画面のUX規約、Googleの同意画面に出るアプリ名の扱い。新しくデモアプリを作るとき、既存アプリの認証を作り直すとき、パスキーやGoogleログインが動かないときに読む。
user-invocable: true
---

# デモアプリの Cognito 認証（おすすめ構成）

デモアプリや Web アプリの認証は、特に指定がなければ「Googleで続ける」かメール（パスキーかメールのワンタイムコード）の構成にし、パスワードは使わない。

動く実物は [minorun365/marp-agent](https://github.com/minorun365/marp-agent) にあるので、迷ったら実物を読む。設計の背景は同リポジトリの `docs/authentication-options.md` に書いてある。

| ファイル | 中身 |
|---|---|
| `infra/lib/auth-stack.ts` | User Pool / App Client / Google IdP / Cognito ドメイン |
| `src/components/Auth/AuthScreen.tsx` | ログイン画面（326行、移植元） |
| `src/components/Auth/AuthScreen.css` | 同スタイル |
| `infra/lambda/auth/google-idp-manager/handler.ts` | Google IdP を作るカスタムリソース |
| `infra/lambda/auth/google-link/handler.ts` | preSignUp で既存メールユーザーへ Google を連携 |
| `infra/lambda/auth/custom-message/handler.ts` | 確認コードとログインのコードのメールを日本語にする |

---

## 1. 利用者から見た形

| 経路 | 初回 | 2回目以降 |
|---|---|---|
| Google | 「Googleで続ける」で登録とログインが同時に終わる | Google |
| メール | メールアドレスだけで登録し、確認コードを入れるとそのままログイン | メールのワンタイムコード（パスキーを登録すれば顔・指紋） |

- パスワードは使わない方針とし、画面に入力欄を置かず、App Client にもパスワードの経路を開けない（下の CDK）。
- メールのコードは Cognito 標準メールで送る。上限は AWS アカウントあたり1日50通で、ログインのたびに1通使うためデモ規模なら収まる。上限に近づいたら SES の本番利用申請を検討する。
- パスキーの登録は任意で、登録しなくてもメールのコードだけで期限なく使える。
- Google 利用者は本人確認を Google 側が担当しているため、パスキー登録を案内しない。

## 2. ログイン画面のUX規約

1画面目は「Googleで続ける」と、メールアドレス入力および「メールで続ける」の2択に絞る。

メールを入力した次の画面では、「パスキーでログイン」と「メールにコードを送る」を常に並べて出す。存在しないアドレスでも Cognito は同じ応答を返すため、コードが来ない人への案内（登録し直しなど）はコード入力の画面に置く。

> パスキー登録の有無で画面を出し分けない。出し分けると「そのメールアドレスが登録済みか」を第三者へ教えることになる。同じ理由で、存在しないアドレスも認証失敗も表示は「メールアドレスまたは認証情報を確認してください。」に統一し、Cognito 側も `preventUserExistenceErrors: true` にする。

メールのコードでログインした利用者には、成功後に一度だけパスキー登録を案内する。ログイン画面を自作するときも、Cognito のマネージドログイン（Hosted UI）に任せるときも同じ扱いで、最初の実装から組み込む。

- 選択肢は「パスキーを登録」と「あとで」の2つにする
- 「あとで」を押されたら30日間は再案内しない（`localStorage` か cookie に押した時刻を持つ）
- 専用の設定画面は作らず、ログイン後のメイン画面も変えない

マネージドログインは、この案内を自分からはほとんど出さない。登録を勧めるのは利用者が自分で新規登録した瞬間だけで、管理者が作ったユーザーや既存ユーザーには何も出ない（AWS 公式ドキュメントに記載）。そのため、次の処理をアプリ側で組む。

- App Client のスコープに `aws.cognito.signin.user.admin`（CDK では `OAuthScope.COGNITO_ADMIN`）を足し、認可要求でも求める
- コードをトークンに替えた直後、本人のアクセストークンで `ListWebAuthnCredentials` を呼び、0件なら案内を出す。IAM の権限は要らず、数えられなかった場合は案内を出さずに通す
- 「登録」を押されたら `https://<Cognitoドメイン>/passkeys/add?client_id=…&redirect_uri=…` へ送る。`response_type` や `state`、PKCE も付けられ、登録後はコードつきで戻る
- Cognito 側のログインは1時間で切れる。切れていると登録画面を出さずに `redirect_uri?result=invalid_session` へ戻るため、受けたらログインし直してから登録画面へ送る

## 3. CDK（User Pool）

パスキーとメールのコードは、設定が1つでも欠けるとエラーが出ないまま使えなくなる。次の5点をまとめて入れる。

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

- ④ の RP ID には、利用者がアクセスするドメインを指定する。`dxxxx.cloudfront.net` のような配信基盤の既定ドメインでも動くが、後からカスタムドメインへ移すと RP ID が変わり、**登録済みのパスキーは全部使えなくなる**。カスタムドメインを付ける予定があるなら、パスキーを入れる前に済ませておく。
- 自動の検査でログインが要る場合は `adminUserPassword: true` だけ足し、`admin-initiate-auth` で入る。AWS の認証情報が要るため、利用者向けの入口にはならない。
- メールを日本語にするには、カスタムメッセージの Lambda トリガーで、確認コード用の `CustomMessage_SignUp` とログインコード用の `CustomMessage_Authentication` の件名と本文を差し替える。パスワードなしのログインも `CustomMessage_Authentication` を通る。
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

`Amplify.configure` の `Auth.Cognito` には `userPoolId` と `userPoolClientId` を渡すだけでよい。Google を使うときだけ `loginWith.oauth` を足す（次節）。

## 4. Google ログイン

### OAuth クライアントはデモごとに作らず、共通の1つを使い回す

同意画面のアプリ名は GCP プロジェクト単位で1つなので、デモごとにクライアントを作っても名前は分けられない。
デモ用の中立な名前でプロジェクトとクライアントを1つ用意し、以後のデモは全部これを使い回す。

| 項目 | 決め方 |
|---|---|
| GCP プロジェクト | デモ専用に1つ作る |
| 同意画面のアプリ名 | 利用者に表示されるため、特定のデモ名にせず中立な名前にする |
| 公開ステータス | 本番環境（テストユーザーの登録なしで誰でもログインできる） |
| Client ID | CDK のコンテキストで渡す（秘密ではない） |
| Client Secret | デプロイ先の AWS アカウントへ置く（下の「client secret の置き場」） |

新しいデモを足すときの作業は次の2つだけになる。

1. Google Cloud Console でそのクライアントを開き、承認済みリダイレクト URI に新しい Cognito ドメインの `/oauth2/idpresponse` を1行足す
2. そのデモの AWS アカウントへ Secret を入れ、CDK に `googleClientId` を渡す

> 組織の Google Workspace 配下の GCP プロジェクトは、同意画面が「内部」（`orgInternalOnly`）になっていることがある。
> その場合は組織のアカウントしかログインできず、外部の人が触るデモでは機能しないため、プロジェクトを選ぶ前に同意画面のユーザータイプを確かめる。
>
> 2026-08 時点では、`gcloud` でも API でもクライアントの作成と編集はできず、リダイレクト URI の追記も含めて Console のブラウザ操作が唯一の手段になる。根拠は3つある。
> - `gcloud alpha iap oauth-clients` は IAP 専用で、ヘルプ自身が「プロジェクト内の全 OAuth クライアントの管理 API としては使えない」と明記している。加えて IAP OAuth Admin API は 2026-03-19 に完全停止している
> - `iap v1` の discovery を見ると `projects.brands.identityAwareProxyClients` の method は `create/get/list/delete/resetSecret` だけで更新系がない。停止していなくても URI は足せない
> - Google の公開 API 一覧（`https://www.googleapis.com/discovery/v1/apis`）に、OAuth クライアントを管理する API 自体が存在しない
>
> 人に操作を頼むときは、クライアントの編集画面まで開ける URL を渡す。URL は次の形になる。
> `https://console.cloud.google.com/auth/clients/<クライアントID>?project=<プロジェクトID>`

### 実装に必要なもの

Hosted UI 用の Cognito ドメイン、Google の OAuth クライアント ID / Secret、IdP 登録、App Client の OAuth 設定を用意する。

```ts
supportedIdentityProviders: [cognito.UserPoolClientIdentityProvider.COGNITO, cognito.UserPoolClientIdentityProvider.GOOGLE],
oAuth: {
  flows: { authorizationCodeGrant: true },
  scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL, cognito.OAuthScope.PROFILE],
  callbackUrls: [`https://${appDomain}/`, 'http://localhost:5173/'],
  logoutUrls:   [`https://${appDomain}/`, 'http://localhost:5173/'],
},
```

フロント側は `signInWithRedirect({ provider: 'Google' })` を呼べばよい。

Amplify Gen2 の `defineAuth` では「スコープ」が2か所にあり、書き方が逆になる。ここを取り違えると、デプロイも構成検査も通ったうえで Google のログイン画面が `invalid_scope` で開かない。

| 場所 | 何のスコープか | 書き方 |
|---|---|---|
| `externalProviders.scopes` | Cognito のアプリクライアントが出すトークンの範囲 | 大文字の列挙（`['EMAIL','PROFILE','OPENID']`） |
| `externalProviders.google.scopes` | Google へそのまま渡る生の文字列 | 小文字（`['openid','email','profile']`） |

後者に大文字を書くと `authorize_scopes` が `"EMAIL PROFILE OPENID"` として IdP に入り、Google が `invalid=[OPENID]` を返す。EMAIL と PROFILE は通ってしまうため、症状が OPENID にしか出ず気づきにくい。

この種の失敗は AWS 側をいくら見ても分からない。デプロイ後に Cognito の認可エンドポイントへリクエストを送り、`accounts.google.com` のログイン画面へ遷移するかを確かめる（サインインは不要）。エラー時は `authError` クエリに base64 で理由が入っている。

```bash
curl -s -o /dev/null -L -w '%{url_effective}\n' \
  "https://<Cognitoドメイン>.auth.<region>.amazoncognito.com/oauth2/authorize?identity_provider=Google&client_id=<アプリクライアントID>&response_type=code&scope=openid+email+profile&redirect_uri=https%3A%2F%2F<配信ドメイン>%2F"
```

`.../signin/oauth/error?authError=...` へ遷移したら失敗している。`authError` を base64url デコードすると、`invalid_scope` や `redirect_uri_mismatch` などの理由が平文で読める。

### client secret の置き場

`cognito.UserPoolIdentityProviderGoogle` の L2 は secret を CloudFormation テンプレートへ平文で残す。marp-agent では、秘密値を state に残さないカスタムリソース（`google-idp-manager`）で IdP を作っている。

> 組織の管理下にある AWS アカウントでは、SCP で `secretsmanager:CreateSecret` が拒否されていることがある。marp-agent の handler は Secrets Manager から読む実装なので、その場合は SSM Parameter Store の SecureString に置き換える。Lambda 実行ロールには `ssm:GetParameter` と、SecureString を復号するための `kms:Decrypt` を付ける。

### 段階的に入れてよい

Google ログインには OAuth クライアント作成や承認済みリダイレクト URI の追加といった GCP 側の作業が要る。クライアント ID が渡されたときだけ Google を有効にする実装にしておくと、パスキーだけ先にリリースできる。marp-agent では `this.node.tryGetContext('googleClientId')` の有無で分岐している。

### 同じメールアドレスの重複プロフィール

Cognito は、メールアドレスが同じでもローカルユーザーと Google ユーザーを自動では統合しない。放置すると同じ人に2つのプロフィールができる。marp-agent では preSignUp トリガー（`google-link`）を使い、確認済みメールアドレスが一致する既存プロフィールへ `AdminLinkProviderForUser` でリンクしている。デモアプリで既存利用者がいないなら、初期は省いてよい。

## 5. デモアプリでの判断（外部の人が触る場合）

- Google の同意画面に出るアプリ名は GCP プロジェクトのブランディングで決まる。クライアント ID を別に作っても同じプロジェクトなら名前は変わらないため、外部の人に見せるデモで無関係な別アプリの名前が出る状態にしない。
- 管理を楽にするなら、デモごとに作るのではなく、デモ共通の中立な名前でプロジェクトとクライアントを1つ用意し、以後のデモはリダイレクト URI を1行足して使い回す。
- セキュリティ要件の厳しい組織の利用者は、業務用 Google アカウントを外部アプリへ繋ぐことに抵抗がある。Google を選択肢に出す場合でも、片方だけにはせずメール経路を同格で並べる。

## 6. 検証（実装完了と言う前に）

パスキーは「画面が出た」だけでは動作確認にならない。次を実操作で確認する。

1. メールで新規登録 → 確認コード → ログイン
2. パスキー登録 → いったんサインアウト → パスキーでログイン
3. パスキー未登録の状態でパスキーを押し、共通エラーが出てメールのコードへ戻れることと、届いたコードのメールが日本語になっていること
4. Google ログイン（有効にした場合）
5. PC と iPhone の両方（iOS Simulator か実機の Safari）

`localhost` は RP ID が `localhost` になるため、本番ドメインで登録したパスキーはローカルでは使えない。ローカルではメールのコードで確認する。画面だけを手元で起動し、本番の User Pool につないで試してもよい。
