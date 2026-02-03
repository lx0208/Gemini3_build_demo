事前にローカル環境では、WIF（Workload Identity Federation）による認証が問題なく動作することを確認しました。

しかし、実際にOCI VM上でプロジェクトを実行したところ、
社内Proxyの制限により、以下の認証用APIへの通信がブロックされていることが判明しました。

・sts.googleapis.com  
・iamcredentials.googleapis.com  

このため、WIFでのトークン交換（access token取得）が失敗し、
Vertex AI（Gemini）APIを呼び出す前の段階で処理が停止してしまいます。

ご提案いただいたPSC（Private Service Connect）について調査した結果、
現在のPSC設定は Vertex AI（aiplatform）API専用であり、
認証処理に必要な STS / IAM API には対応していないことが分かりました。

そのため、PSC経由のみではWIF認証は成立せず、
本件は解決できない状況です。

対応策としては、以下のいずれかが必要と考えております。

・Proxyのホワイトリストに  
　sts.googleapis.com  
　iamcredentials.googleapis.com  
　を追加する  
または  
・認証用APIも含めたPSC／通信許可の設計を行う

現状の構成を最小限変更する方法としては、
STS / IAM API のホワイトリスト追加が最も現実的だと考えております。

ご確認・ご検討のほど、よろしくお願いいたします。


----


補足として、PSC（Private Service Connect）利用時の認証方式について説明いたします。

現在ご提供いただいているPSC構成では、
以下の認証方式のみが前提となっていると理解しております。

・ADC（Application Default Credentials）による
　GCP VPC内からの認証
・Service Account Key（鍵ファイル）を用いた認証

しかし、本案件の構成ではOCI VM上で実行しており、
GCP VPC内のADCは利用できないため、
外部環境向けの認証方式としてWIF（Workload Identity Federation）を採用しております。

また、Service Account Key認証については、
セキュリティポリシー上、原則として使用しない方針となっております。

理由としては以下の通りです。

・長期間有効な秘密鍵ファイルの管理リスクが高い  
・鍵漏洩時の影響範囲が大きい  
・クラウドベンダー（Google）公式としても非推奨である  

そのため、
「ADC（GCP内）または Service Account Key 前提」
というPSC設計では、
OCIなどの外部環境＋WIF構成では成立しない状況となります。

外部環境から安全にVertex AIを利用するためには、
WIF認証に必要な以下のAPI通信が必須となります。

・sts.googleapis.com  
・iamcredentials.googleapis.com  

これらをProxyのホワイトリストに追加していただくことで、
Service Account Keyを使用せず、
セキュリティを担保した形での連携が可能となります。
