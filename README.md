# Gemini3_build_demo
Gemini 3 vibe code
sequenceDiagram
    autonumber
    participant VM as OCI VM (Python App)
    participant KC as Keycloak (OIDC Provider)
    participant STS as GCP STS (sts.googleapis.com)
    participant DISC as Keycloak OIDC Discovery/JWKS
    participant IAMC as IAMCredentials (generateAccessToken)
    participant VAI as Vertex AI API (aiplatform.googleapis.com)

    Note over VM: 0) 预置：Keycloak client(Confidential + Service accounts)\n已配置 aud（Audience mapper）\nGCP 已配置 WIF Pool/Provider + SA 绑定权限

    VM->>KC: 1) POST /realms/<realm>/protocol/openid-connect/token\n grant_type=client_credentials\n client_id + client_secret
    KC-->>VM: 2) access_token (JWT)\n iss=<realm issuer>\n sub=<client/service-account subject>\n aud=<你配置的aud>\n exp=...

    Note over VM: 3) VM 把 JWT 作为 subject_token\n（写文件或直接内存传给 google-auth external_account）

    VM->>STS: 4) Token Exchange (RFC8693)\n subject_token=JWT\n subject_token_type=jwt\n audience=//iam.googleapis.com/.../workloadIdentityPools/.../providers/...
    Note over STS: 5) STS 需要验证 JWT 签名与声明

    STS->>DISC: 6) GET /.well-known/openid-configuration
    DISC-->>STS: 7) 返回 jwks_uri / issuer 等信息
    STS->>DISC: 8) GET <jwks_uri> 拉取公钥集合(JWKS)
    DISC-->>STS: 9) 返回公钥(KID->Key)

    Note over STS: 10) 校验：iss/aud/exp/签名(KID)\n通过后生成 federated token
    STS-->>VM: 11) 返回 federated access token（短期）

    alt 使用 Service Account Impersonation（推荐）
        VM->>IAMC: 12) generateAccessToken\n Authorization: Bearer <federated token>\n 目标：serviceAccount:<SA_EMAIL>
        IAMC-->>VM: 13) 返回 SA access token（更标准/权限明确）
        VM->>VAI: 14) 调用 Vertex AI\n Authorization: Bearer <SA access token>
        VAI-->>VM: 15) 响应
    else 不做 Impersonation（不常用）
        VM->>VAI: 14') 直接用 federated token 调用（取决于配置）
        VAI-->>VM: 15') 响应
    end

    Note over VM: 16) Token 过期前刷新：\n重新向 Keycloak 取 JWT → STS exchange → (可选) impersonate → 调用 API
