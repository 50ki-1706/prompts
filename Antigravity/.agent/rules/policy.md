## Language
- ImplementationPlan も含めて日本語で応答する

## File Access Policy
以下のファイル・パスは読み取り・参照・要約・内容推測を行わない。

### Environment / Local Settings
**/.env
**/.env.*
**/*.env
**/*.env.*
**/*.local.*

### Keys / Certificates
**/id_*
**/*.pem
**/*.key
**/*.p12
**/*.pfx
**/*.crt

### Cloud / Service Credentials
**/service-account*.json
**/*credentials*
**/.aws/**
**/.azure/**
**/.config/gcloud/**

### Token-related Files
**/.npmrc
**/.pypirc
**/.netrc
**/docker/config.json

### CI / Infrastructure State
**/.github/workflows/**
**/*ci*.yml
**/terraform.tfstate
**/*.tfstate*

### Sessions / Logs / Dumps
**/cookies.txt
**/*.log
**/*.har
**/*.dump
**/core.*

### Backups / Old Data
**/*.bak
**/*.old
**/*~

---

## Command Safety Policy
以下のコマンドは自動実行しない。
実行を伴う場合、事前に「実行内容」「影響範囲」「リスク」を説明する。

### Privilege / User
sudo
su
chmod
chown
userdel
groupdel
usermod
passwd

### File / Disk
rm
dd
mkfs
mount
umount
mv
cp

### Git
git reset
git clean
git push
git checkout --
git rebase
git reflog expire
git gc

### Network / Transfer
curl
wget
ssh
scp
rsync

### Package Managers (OS Level)
apt
apt-get
yum
dnf

### System Control
shutdown
reboot
kill
killall
systemctl

### Container / Virtualization
docker
docker-compose
docker compose
kubectl

### Infrastructure as Code
terraform apply
terraform destroy
pulumi up
pulumi destroy

### Cloud CLI
aws
gcloud
az

### Database / Data Migration
mysql
psql
mongo
redis-cli
rails db:migrate
rails db:reset
prisma migrate

---

## Package Installation Policy
以下のコマンドは「外部パッケージの導入」または
「外部コードの実行」を伴うため、自動実行しない。
実行前に「導入内容」と「リスク」を説明する。
下記に記載のないインストール系のコマンドも同様に説明を求める。

### Node.js
npm install
npm i
npm add
npm ci
npx

### Python
pip install
pip3 install
poetry add
poetry install
pipenv install

### Ruby
gem install
bundle install

### PHP
composer install
composer require

### Rust
cargo install
cargo add

### Go
go install
go get

### Other
conda install
pnpm install
yarn add
brew install
brew upgrade
brew uninstall

---

## Execution Rules
- 危険操作は説明なしで実行しない
- 不可逆操作は特に注意を促す
- 不可逆操作とは、データ・履歴・権限・インフラ状態を元に戻せない操作を指す
- `--force`, `-f`, `-r`, `-a`, `--hard` を含む操作は危険度を一段引き上げる
- `curl | sh` や `wget | bash` は外部コード即時実行として特に注意する
- npx は一時的な外部コード実行を伴うため特に注意する
- 曖昧な指示・対象が不明な場合は実行せず確認を求める
