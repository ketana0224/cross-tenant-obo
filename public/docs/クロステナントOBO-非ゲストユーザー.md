# クロステナントOBO-非ゲストユーザー

[ゲストユーザー版](クロステナントOBO-ゲストユーザー.md)と同じ配置で、M365テナントの利用者をAzureテナントに登録しない場合を説明する。

**Foundryと下流サービスの認証先をAzureテナントのままにする場合、ゲスト招待だけを省いても、ゲストユーザー版と同じOBOは成立しない。**

## 1. 今回の構成とOBO

| 対象 | 所属・配置先 |
| --- | --- |
| 利用者 | M365テナント。Azureテナントには利用者登録がない |
| Foundryエージェント | Azureテナント |
| 下流サービス | Foundryと同じAzureテナント |

ここでは、次を前提とする。

- Foundryと下流サービスの認証先、および下流APIを表すアプリの登録先はAzureテナントとする。
- 「非ゲスト」は、ゲスト招待だけでなく、ほかの方法によるAzureテナントへの利用者登録も行わない状態を指す。
- 利用者本人のトークンでFoundryを直接呼び出す。エンドポイントは `AgentBlueprint` 方式で、受け入れるテナントはAzureテナントとする。
- 下流には、Azureテナントの本人を表す委任トークンでアクセスする。ツール接続は `AgentOnBehalfOfUser` とする。これらのFoundry機能はプレビューである。

```text
M365テナントの利用者
    ↓ Azureテナントには利用者登録がない
AzureテナントのFoundry
    × Azureテナントの本人としてのアクセスが成立しない
Azureテナントの下流サービス
    本人の代理アクセスには到達しない
```

M365のアカウントにサインインできることと、Azureテナントの本人としてトークンを取得できることは別である。

### トークン取得から処理結果までの流れ

ゲストユーザー版と同じ①〜⑨を使い、どこで先に進めなくなるかを示す。

```mermaid
sequenceDiagram
    actor User as 利用者<br/>M365テナント所属
    participant Client as クライアントアプリ
    box rgb(235, 243, 255) Azureテナント
        participant Foundry as Foundryエージェント
        participant Entra as Microsoft Entra ID
        participant API as 下流サービス<br/>API／MCPサーバー／A2Aエージェント
    end

    User->>Client: ① 利用開始

    alt Azureテナントで入口トークンAを取得しようとする
        rect rgb(235, 243, 255)
            Note over Client,Entra: ② 入口トークンAの取得
            Client->>Entra: Foundryエージェント宛ての本人トークンを要求
            User->>Entra: M365のアカウントでサインイン
            Entra--xClient: Azureテナントに本人が未登録のため、Aの取得に進めない
        end
    else M365テナント発行の本人トークンで代用しようとする
        Client->>Foundry: ③ M365テナントの本人トークンを付けて処理を依頼
        Foundry->>Foundry: ④ トークンの発行元・宛先・本人・権限を確認
        Foundry--xClient: Azureテナントのトークンを受け入れる条件を満たさず拒否
    end

    rect rgb(255, 246, 228)
        Note over Foundry,Entra: ⑤〜⑥ OBOによるトークン交換<br/>この依頼では実行に進まない
    end

    rect rgb(235, 248, 238)
        Note over Foundry,API: ⑦〜⑧ 下流サービスの呼び出し<br/>この依頼では実行に進まない
    end

    Note over Client,API: ⑨ 下流の処理結果は得られない
```

### トークンAとBは誰を表すか

| 項目 | ゲストユーザー版 | この構成 |
| --- | --- | --- |
| Azureテナントの本人 | ゲストとして登録されている | 登録されていない |
| Azureテナント発行の入口トークンA | Azureテナントのゲスト本人を表す | 同じ本人トークンを取得する前提を満たさない |
| M365テナント発行の本人トークン | Azureテナント発行のAの代わりには使わない | Azureテナントの本人登録や、入口の受け入れ条件を代用しない |
| Azureテナント発行の下流トークンB | 同じゲスト本人を表す | この依頼からは取得に進まない |

**OBOは、M365テナントの本人を自動的にAzureテナントの本人へ変換する仕組みではない。**

**不成立という結論は、Azure側に本人登録がなく、入口・下流の認証先をAzureテナントに固定した本書の前提に限る。非ゲストのすべての構成でOBOが不成立という意味ではない。**
