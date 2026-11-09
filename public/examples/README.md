# 匿名化した接続設定テンプレート

以下は**接続propertiesの概念例**であり、デプロイスクリプトや検証済みの完成パッケージではない。ID・URL・秘密はすべてプレースホルダー。実環境に送信する前に、その時点のAPIバージョン・必要な設定・権限を確認する。

| ファイル | 接続方式 | 下流主体 |
| --- | --- | --- |
| [foundry-mcp-obo.connection.json](foundry-mcp-obo.connection.json) | RemoteTool／AgentOnBehalfOfUser | 本人に代わるエージェント |
| [foundry-a2a-obo.connection.json](foundry-a2a-obo.connection.json) | RemoteA2A／AgentOnBehalfOfUser | 本人に代わるエージェント |
| [foundry-a2a-oauth.connection.json](foundry-a2a-oauth.connection.json) | RemoteA2A／OAuth2 | 下流用OAuthで認証した本人 |
| [foundry-mcp-mi.connection.json](foundry-mcp-mi.connection.json) | RemoteTool／ProjectManagedIdentity | プロジェクトMI |

## プレースホルダー

- `<API_APP_ID>`：下流リソースアプリのApp ID。
- `<API_TENANT_ID>`：OAuth下流が認証するテナント。ホスト場所で決めない。
- `<OAUTH_CLIENT_ID>`：下流用OAuthアプリ。
- `<SET_PRIVATELY_DO_NOT_COMMIT>`：実際の資格情報は非公開の設定／資格情報ストア側で管理し、公開ファイルへ書き込まない。
- `*.example.invalid`：実環境ではない例示URL。実URLは非公開設定で保持する。

OAuth例の `clientSecret` は説明用プレースホルダーであり、secretRef機能の実装例ではない。秘密参照や安全な投入方式は利用する管理ツールで別途設計する。認証プロバイダーのリダイレクトURI等も別途設定が必要。

## テンプレートだけでは成立しない

**OBO**：Blueprint入口、Blueprint宛の本人トークン、入口利用権限、下流の委任権限・同意、下流のJWT検証／本人認可が必要。

**OAuth2**：下流アプリ、認可／トークンURL、許可されたリダイレクトURI、同意、本人の初回サインイン・MFA／許可、同じ実行の再開が必要。

**MI**：対象APIのaudience、アプリロール割当、下流が受け入れるMI・発行元、実操作／データ認可が必要。

スコープ名 `session:scope:analyst` は今回の疑似APIの例。任意の実APIに共通のスコープではない。`/.default` を任意の接続へ機械的に置き換えない。

詳しくは[成立条件](../docs/認証方式と成立条件.md)・[検証方法](../docs/検証方法と限界.md)・[公開ポリシー](../docs/公開と非公開のポリシー.md)。
