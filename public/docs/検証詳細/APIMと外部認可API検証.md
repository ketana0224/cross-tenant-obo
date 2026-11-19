# APIM・本人OBO・外部認可APIの検証詳細

検証日：**2026-11-12（JST）**。[詳細の入口](README.md)／[Copilotチャネルの詳細](CopilotStudioとM365Copilot検証.md)。

## 1. 結論と実装境界

Developer SKUのAPIMを明示的にMCP接続先に指定し、次の3段階を検証した。

| 公開ID | 段階 | 実機記録 |
| --- | --- | --- |
| P01 | アプリロール・スコープ・クライアント検査、プロジェクトMI | 16／16ケースで期待値と一致。直接HTTP試験を含む |
| P02 | Foundryの本人OBOとAPIM内認可 | 3／3ケースで期待値と一致 |
| P03 | Foundryの本人OBOと外部認可API | 7／7ケースで期待値と一致 |

拒否を期待した試験は、拒否を確認して成功とする。**全ケースがツール実行を許可されたという意味ではない。** 各段階は異なる構成・時点であり、後日の一般利用者対応後にすべて再実行した結果ではない。

APIMの一般的なJWT・クレーム検査を使った検証であり、FoundryポータルのAI Gateway自動統合やAgent 365連携の実機検証ではない。

## 2. 構成とトークン

```text
本人の対話サインイン
  → Blueprint向け入口トークン（scp=access_agent）
  → FoundryのAgentBlueprint入口
  → AgentOnBehalfOfUser接続
  → API向け委任トークン
  → APIMの署名・発行元・期限・audience・scope検証
      ├─ P02：APIM内でツール別判定 → 合成MCP応答
      └─ P03：send-request → 認可API
                  ├─ 許可 → 合成MCPバックエンド
                  └─ 拒否／障害 → 転送しない
```

| 項目 | Foundry入口 | APIM到着 |
| --- | --- | --- |
| `aud` | 専用Blueprint | 検証API |
| `scp` | `access_agent` | `Probe.Read` |
| `oid` | サインインした本人 | 同じ本人 |
| 呼び出し元 | 専用サインインクライアント | 対象エージェントID |

下流トークンは検証スクリプトが注入したものではなく、Foundryが取得した。APIMの検証済み到着claimsとnative実出力を確認したが、Entraトークンエンドポイントの通信自体はキャプチャしていない。

P01のMIでは本人の委任スコープではなく、APIアプリロール`Tools.Read`を付与したプロジェクトMIを確認した。`Tools.Read`と`Probe.Read`は異なる権限種別であり混同しない。

## 3. P01：MIを含む16ケース

この段階のAPI／MCP応答はAPIM内の合成データで、外部バックエンドへの転送はない。

| # | 操作・条件 | 観測 |
| --- | --- | --- |
| 1 | トークンなし | HTTP 401 |
| 2 | 不正トークン | HTTP 401 |
| 3 | `Tools.Read`でread API | HTTP 200、allowed |
| 4 | `Tools.Read`だけでwrite API | HTTP 403 |
| 5 | ロールなしでread API | HTTP 403 |
| 6 | 許可クライアントIDと不一致 | HTTP 401 |
| 7 | 許可クライアントIDと一致 | HTTP 200、allowed |
| 8 | `required-claims`の`Tools.Read`あり | HTTP 200、allowed |
| 9 | 同ロールなし | HTTP 401 |
| 10 | 別操作で委任スコープ`user_impersonation`を検査 | HTTP 200、allowed |
| 11 | 未付与の`Not.Granted`を要求 | HTTP 401 |
| 12 | ARM向けトークンで合成APIを呼ぶ | HTTP 401 |
| 13 | アプリからMCPの`read_probe` | HTTP 200、`isError=false`、allowed |
| 14 | 同じアプリから`write_probe` | HTTP 200、`isError=true`、denied |
| 15 | FoundryのMIから`read_probe` | native MCP出力でMI・API宛先・ロール・allowedを照合 |
| 16 | 同じMIから`write_probe` | native MCP出力でMI・API宛先・ロール・deniedを照合 |

ケース10・11はARM向け委任トークンを別操作で検査したもので、Foundryの本人OBO試験ではない。ケース12のAPI宛先不一致拒否とも区別する。

## 4. P02：本人OBOとAPIM内認可

| # | 操作 | 観測 |
| --- | --- | --- |
| 1 | Blueprint向け入口トークンを交換せずAPIMに直接送る | audience不一致で401 |
| 2 | native OBOで`read_probe` | 実MCP出力で本人・API宛先・`Probe.Read`・allowedを確認 |
| 3 | 同じ本人・スコープで`write_probe` | 実MCP出力でdenied、`isError=true` |

書き込み用`Probe.Write`は公開・付与していない。拒否はMCPエラーとして配送されるため、HTTP 200やFoundryの`completed`だけで許可と判断しない。

## 5. P03：外部認可API

### APIMと認可APIの責務

1. APIMの`validate-azure-ad-token`でJWTを検証する。
2. 検証済みJWTから`tid`・`oid`・`aud`・`scp`・呼び出し元を抽出する。呼び出し元は`azp`、なければ`appid`を使う。利用者の任意ヘッダーを本人証明にしない。
3. MCP本文のメソッド・ツール名、API／操作、HTTPメソッド、パス、要求相関IDとともに`send-request`で認可APIへ送る。
4. HTTP成功に加え、厳密なbooleanの`allowed`、一意な判定ID、元と一致する要求相関IDを確認する。
5. 許可時だけ元のAPI向けOBOトークンを保持して合成MCPへ転送する。拒否は403、障害・期限超過・不正応答は503で閉じる。

認可APIにはトークン本体ではなく検証済みclaimsと対象情報だけを送った。APIMから認可APIへの認証には専用秘密キーを使用し、secret Named valueとContainer Apps Secretで管理した。

外部認可版のwrite拒否は、同じ`Probe.Read`でもツール別ACLにより認可APIが拒否する構成。APIM内認可版の`Probe.Write`不足とは判定場所・規則が異なる。

### 実機結果

| # | 条件 | APIM／認可APIの観測 | 対応ツールのバックエンド実行 |
| --- | --- | --- | --- |
| 1 | 専用資格情報なしで認可APIへ直接アクセス | 認可APIが401 | なし |
| 2 | 入口トークンをAPIMへ直接送る | APIMが401 | なし |
| 3 | 本人OBOで`read_probe` | allowed=true、native MCP出力と照合 | 1回 |
| 4 | 同じ本人で`write_probe` | allowed=false、APIMは403 | 0回 |
| 5 | 認可APIが503 | APIMも503、許可へ切り替えない | 0回 |
| 6 | 認可APIが4秒待機、APIM期限2秒 | APIMは503 | 0回 |
| 7 | 認可APIが文字列`"allowed":"true"`を返す | 型不正としてAPIMは503 | 0回 |

Foundryでは拒否・障害が`tool_user_error`／HTTP 400として通知された。これはAPIMの403／503とは別の層のコード。

Azure Monitorでも対象時間帯の401×1、403×1、503×3（うちTimeout×1）、いずれも`BackendResponseCode=0`を確認した。この0は主バックエンドへの未転送を意味し、認可APIへの`send-request`が呼ばれていないという意味ではない。個別の未転送は相関ID付き監査と照合した。

## 6. 遅延・キャッシュ・実行上の注意

| 経路 | read時の認可往復時間（各1サンプル） |
| --- | --- |
| Foundry | 111.8733 ms |
| Copilot Studio・管理者 | 124.1242 ms |
| M365 Copilot・管理者 | 119.1162 ms |
| M365 Copilot・一般利用者 | 106.6254 ms |

APIMの`send-request`前後差分であり、通信を含む。平均、上限、SLA、エージェント全体の応答時間ではない。Foundryの当該1サンプルは全体で約17.14秒だった。

- 認可はツール本体だけでなく、MCP初期化・探索・一覧にも発生する。
- 認可APIは最小0／最大1レプリカ。コールドスタートで期限2秒を超え503となる事象を観測し、起動確認後の試行で成功した。常時可用性の検証ではない。
- DB・Redis・認可結果キャッシュは未実装・未検証。`cache-lookup-value`／`cache-store-value`も設定していない。
- APIMからのチャンク転送未対応時にバックエンド413を観測し、対応後に解消した。クライアントの汎用エラーだけでは原因を特定できない。
- Container AppsのSecret更新だけで稼働コンテナに反映されたと判断せず、新revisionと認証付きヘルス確認を使った。

## 7. 成功から一般化しないこと

- 実データの行／文書別認可、DB更新、本番のバックエンド直アクセス防止は対象外。
- 合成MCPはAPIMの署名検証と専用キーによる信頼ホップを前提にclaimsを参照する。バックエンドでの独立JWT署名検証は未検証。
- 認可APIと合成MCPは同一Container Appの別エンドポイント。独立した障害ドメインではない。
- 一般利用者対応後のFoundry全実機回帰は未実施。最終25件の単体テスト記録は過去の3／7ケース再実行の代用ではない。
- このAPIM系列でのFoundry一般利用者、クロステナントOBO、全トークン境界条件、負荷、プライベートネットワークは未検証。

追試の判定と記録は[再検証手順](再検証手順と記録項目.md)、仕様の参照先は[公式資料](../Preview/公式資料と参照一覧.md)を参照。
