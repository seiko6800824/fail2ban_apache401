# fail2ban Lab Writeup: Apache BasicAuth 401 Jail（2025-11-23）

## 目的
Apache の Basic認証（`/secret/`）に対する認証失敗（401）を検知し、fail2ban で自動BAN→解除までを再現する。

---

## 実施環境
- Ubuntu Server（VirtualBox）
- Apache2 + BasicAuth（/secret/）
- fail2ban（nftables backend）

---

## 設定ファイル

### 1) Jail 設定
**/etc/fail2ban/jail.d/apache-401.local**
```ini
[apache-401]
enabled  = true
port     = http,https
filter   = apache-401
logpath  = /var/log/apache2/access.log

backend  = polling
banaction = nftables-multiport

maxretry = 3
findtime = 10m
bantime  = 30m
```

### 2) Filter 設定
**/etc/fail2ban/filter.d/apache-401.conf**
```ini
[Definition]
failregex = ^<HOST> .*"(GET|POST) /secret/? HTTP/.*" 401
ignoreregex =
```

---

## 反映手順
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client reload
sudo fail2ban-client status
sudo fail2ban-client status apache-401
```

---

## テスト方法（ブラウザ）
1. Windows 側ブラウザで `http://<VM_IP>/secret/` にアクセス  
2. Basic認証のユーザー/パスを **わざと間違えて3回** 送信  
   - 目的：401 を連続発生させて検知させる

---

## 結果
```bash
sudo fail2ban-client status apache-401
- `Total failed: 3` を検知
- `Currently banned: 1`
- `Banned IP list: <YOUR_CLIENT_IP>`  
  → **401連打によりBAN成立**
- apache-401 jail が有効化され、/secret/ への認証失敗(401)を検知。
- maxretry=3 に調整後、クライアントIP(192.168.56.1)がBANされたことを確認。
- unbanip コマンドで解除できることも確認。

### 確認ログ（抜粋・テキストのみ）
- `fail2ban-client status apache-401`
  - Total failed: 3
  - Currently banned: 1
  - Banned IP list: 192.168.56.1
- `fail2ban-client set apache-401 unbanip 192.168.56.1`
  - Currently banned: 0

---

## 解除
```bash
sudo fail2ban-client set apache-401 unbanip <YOUR_CLIENT_IP>
sudo fail2ban-client status apache-401
```

- `Currently banned: 0`
- `Banned IP list:` 空  
  → **解除成功**

※ `Total banned` は「累計BAN回数」なので 0 に戻らなくてOK。

---

## 学び（超要点）
- 401 は「認証突破（ブルートフォース）っぽい挙動」として自然な検知対象。
- `maxretry=5` だとBANが見えにくいため **3に下げると再現が速い**。
- `backend=polling` ＋ `banaction=nftables-multiport` で反映が安定。

---

## 次の一歩（任意）
- 404（偵察/ディレクトリ総当たり）用 jail を追加して  
  「認証突破対策」と「偵察対策」を並べたWriteupにすると実務っぽい。
