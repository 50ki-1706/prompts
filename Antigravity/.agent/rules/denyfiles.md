## File Access Policy

以下のファイル・パスは読み取り・参照・要約・内容推測を行わない。

### Environment / Local Settings

**/.env
**/.env._
\*\*/_.env
**/_.env._
**/_.local._

### Keys / Certificates

id\_\*
_.pem
_.key
_.p12
_.pfx
\*.crt

### Cloud / Service Credentials

**/service-account\*.json
**/_credentials_
**/.aws/\*
**/.azure/_
\*\*/.config/gcloud/_

### Token-related Files

.npmrc
.pypirc
.netrc
\*\*/docker/config.json

### CI / Infrastructure State

**/.github/workflows/\*
**/_ci_.yml
terraform.tfstate
_.tfstate_

### Sessions / Logs / Dumps

cookies.txt
_.log
_.har
_.dump
core._

### Backups / Old Data

_.bak
_.old
\*~