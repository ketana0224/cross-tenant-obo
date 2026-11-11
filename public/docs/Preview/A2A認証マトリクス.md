# Foundry -> A2Aの認証マトリクス

検証日：**2026-11-09**。原資料のFoundry 7〜22行をA2Aに置換した16構成に `T` を付けた。

すべて専用A2A identity agentへのnative呼び出し。元のSharePoint・MCP・OpenAPIの仕様やライセンス条件をA2Aへ転用しない。CSからのA2A、Foundry-to-Foundryのすべての組合せ、マルチテナントAPI追加構成は未検証。

## 判定

- **○**：最終応答が完了し、native A2A出力・期待するトークン識別情報と権限・同じ実行の受信側生成receiptとログを照合。
- **×**：その構成の実行不成立。タイムアウトは製品非対応の確定根拠ではない。
- **—**：匿名負対照は本人OBO判定対象外。401を本人OBOの×にしない。
- 個別OAuth・MIの○は本人OBOの成功ではない。

## 最新の16構成

| ID | Foundry配置 | 入口 | A2Aホスト | 下流発行元 | native下流認証 | 判定 | 受信側／最終応答の確認 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| T07 | R | Rゲスト本人 | H | H | AgentOnBehalfOfUser | × | SendMessageタイムアウト。本人受信は未確認 |
| T08 | R | Rゲスト本人 | R | H | 匿名負対照 | — | A2Aが401拒否。Foundry側400。本人OBOの試験ではない |
| T09 | R | Rゲスト本人 | R | H | AgentOnBehalfOfUser | × | SendMessageタイムアウト。本人受信は未確認 |
| T10 | R | Rゲスト本人 | R | H | OAuth2 | ○ 個別OAuth | H本人の受信と最終応答を照合。入口からの本人OBOではない |
| T11 | R | Rゲスト本人 | R | R | ProjectManagedIdentity | ○ MI | RプロジェクトMI・アプリ権限と最終応答を照合 |
| T12 | R | Rゲスト本人 | R | R | AgentOnBehalfOfUser | ○ 本人OBO | Rゲスト本人・委任権限と最終応答を照合 |
| T13 | R | Rゲスト本人 | R | R | OAuth2 | ○ 個別OAuth | 下流用Rサインイン後、Rゲスト本人と最終応答を照合 |
| T14 | R | Rゲスト本人 | R | R | AgentOnBehalfOfUser | ○ 本人OBO | Rゲスト本人・委任権限と最終応答を照合。SharePoint試験ではない |
| T15 | H | H本人 | H | H | AgentOnBehalfOfUser | ○ 本人OBO | H本人・委任権限と最終応答を照合。SharePoint試験ではない |
| T16 | H | H本人 | R | H | 匿名負対照 | — | A2Aが401拒否。Foundry側400。本人OBOの試験ではない |
| T17 | H | H本人 | R | H | AgentOnBehalfOfUser | ○ 本人OBO | H本人・委任権限と最終応答を照合。別ホストでも成立 |
| T18 | H | H本人 | R | H | OAuth2 | ○ 個別OAuth | 下流用Hサインイン後、H本人と最終応答を照合 |
| T19 | H | H本人 | R | R | ProjectManagedIdentity | × | ARA BadRequestでトークン取得失敗。MI受信は未確認 |
| T20 | H | H本人 | R | R | AgentOnBehalfOfUser | × | SendMessageタイムアウト。本人受信は未確認 |
| T21 | H | H本人 | R | R | OAuth2 | ○ 個別OAuth | 下流用Rゲストサインイン後、本人と最終応答を照合 |
| T22 | H | H本人 | R | R | AgentOnBehalfOfUser | × | SendMessageタイムアウト。本人受信は未確認。SharePoint試験ではない |

集計：**○9件（本人OBO4・個別OAuth4・MI1）、×5件、匿名負対照2件。** 全16構成を実行したが、匿名負対照をOBO成立／非成立の件数には含めない。

## 共通設定と成立条件

- 実機は `a2a_preview` ツールとA2A 0.3系の専用identity agentを使用。A2A 1.0の `a2a` ツールへ結果を流用しない。詳しくは[A2Aの基本](A2Aの基本.md)を参照。
- A2Aサーバーの認証用Entraアプリは、受信側の認証先テナントへのシングルテナント登録。本人OBO／個別OAuth用の委任スコープと、プロジェクトMI用のアプリロールを公開した。個別OAuthではOAuthクライアントアプリも別途用意した。
- OBO入口は単一テナントのAgentBlueprint。許可する本人・クライアント・テナント・宛先を限定した。
- OBO接続は `RemoteA2A`／`AgentOnBehalfOfUser`、下流APIの完全修飾委任スコープ。
- OAuth2接続は下流テナントの本人サインイン。初回の通常OAuth接続を別途事前作成して代用していない。
- MI接続はプロジェクトMIと対象APIのaudience。本人の委任トークンとは区別した。
- 下流APIは元の単一テナント登録を保持。MCPの[D09-M・D20-M](本人委任_API_MCPマトリクス.md)の追加構成とは別試験。
- エンドポイント稼働確認のHTTP 200、生成された回答、OAuth同意画面だけでは○にしていない。

詳しくは[成立条件](認証方式と成立条件.md)・[設定例](../../examples/README.md)。

## 再検証で解消した実行阻害要因

1. native OAuthの `consent_link` が `OAuth consent required. Please visit: ` という固定メッセージ付きだった。固定接頭辞を除いてHTTPS・許可ドメインを検証するようUIを修正し、本人の認証／許可後に同じ実行を再開した。
2. R側モデルが1要求／分・1,000トークン／分で、受信後のモデル処理が429になった。承認・空きクォータ確認後に10要求／分・10,000トークン／分へ増やし、R側8構成を再実行した。モデル・認証境界は保持。

容量は検証環境の条件であり、OBOの認証成立条件や、推奨される最低容量を示すものではない。変更前の受信確認と429だけを、最終応答までの○にはしていない。

## ここから言えること／言えないこと

**言えること**：ホストが別テナントでも、入口の本人と下流の発行元・宛先・委任権限が整合した構成で本人OBOは成立した。

**言えないこと**：タイムアウトの全原因、A2AでのマルチテナントAPI追加構成の可否、すべてのA2Aサーバー／チャネルの互換性、業務データの認可、実Snowflakeへの適用。
