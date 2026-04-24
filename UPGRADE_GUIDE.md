# Huong dan nang cap WSO2 Identity Server len 7.2.0 GA

## Muc luc

1. [Yeu cau he thong](#1-yeu-cau-he-thong)
2. [Chuan bi truoc khi nang cap](#2-chuan-bi-truoc-khi-nang-cap)
3. [Cac buoc nang cap](#3-cac-buoc-nang-cap)
4. [Cap nhat Database Schema](#4-cap-nhat-database-schema)
5. [Cap nhat Configuration](#5-cap-nhat-configuration)
6. [Chay Migration Client](#6-chay-migration-client)
7. [Khoi dong va kiem tra](#7-khoi-dong-va-kiem-tra)
8. [Rollback plan](#8-rollback-plan)

---

## 1. Yeu cau he thong

| Thanh phan | Yeu cau |
|------------|---------|
| JDK | 11, 17, hoac 21 (khuyen nghi JDK 17+) |
| Database | MySQL 8.x / PostgreSQL 14+ / Oracle 19c+ / MSSQL 2019+ |
| RAM | Toi thieu 2GB, khuyen nghi 4GB+ |
| Disk | Toi thieu 1GB free cho ban cai dat |
| OS | Linux (khuyen nghi), Windows Server, macOS (dev) |

---

## 2. Chuan bi truoc khi nang cap

### 2.1 Backup toan bo

```bash
# 1. Backup thu muc IS hien tai
cp -r <IS_HOME_7.0.0> <IS_HOME_7.0.0>_backup_$(date +%Y%m%d)

# 2. Backup database
# MySQL
mysqldump -u wso2carbon -p WSO2IDENTITY_DB > identity_db_backup.sql
mysqldump -u wso2carbon -p WSO2SHARED_DB > shared_db_backup.sql

# PostgreSQL
pg_dump -U wso2carbon WSO2IDENTITY_DB > identity_db_backup.sql
pg_dump -U wso2carbon WSO2SHARED_DB > shared_db_backup.sql

# 3. Backup keystore va truststore
cp <IS_HOME>/repository/resources/security/wso2carbon.p12 ./keystores_backup/
cp <IS_HOME>/repository/resources/security/client-truststore.p12 ./keystores_backup/

# 4. Luu lai danh sach custom configurations
diff <IS_HOME>/repository/conf/deployment.toml <IS_ORIGINAL>/repository/conf/deployment.toml > custom_config.diff
```

### 2.2 Ghi lai cac tuy chinh hien tai

Liet ke tat ca nhung gi da thay doi trong ban 7.0.0:
- [ ] Custom authenticators (JARs trong `dropins/` hoac `lib/`)
- [ ] Custom event handlers
- [ ] Custom user store managers
- [ ] Thay doi trong `deployment.toml`
- [ ] Custom themes/branding
- [ ] Custom REST API extensions
- [ ] Custom adaptive authentication scripts
- [ ] Third-party connectors
- [ ] Custom SQL scripts

### 2.3 Kiem tra tuong thich

- Kiem tra cac custom JAR co tuong thich voi Carbon Identity Framework 7.6.x+
- Kiem tra cac connector/authenticator cua ben thu ba co ho tro IS 7.2.0
- Review release notes: https://github.com/wso2/product-is/releases

---

## 3. Cac buoc nang cap

### Buoc 1: Tai ban IS 7.2.0 GA

```bash
# Tai tu trang chinh thuc
# https://wso2.com/identity-server/
# Giai nen
unzip wso2is-7.2.0.zip -d /opt/
export NEW_IS_HOME=/opt/wso2is-7.2.0
export OLD_IS_HOME=/opt/wso2is-7.0.0
```

### Buoc 2: Chuyen cau hinh tu ban cu

```bash
# Sao chep deployment.toml tu ban cu
cp $OLD_IS_HOME/repository/conf/deployment.toml $NEW_IS_HOME/repository/conf/deployment.toml

# Sao chep keystore va truststore
cp $OLD_IS_HOME/repository/resources/security/wso2carbon.p12 \
   $NEW_IS_HOME/repository/resources/security/
cp $OLD_IS_HOME/repository/resources/security/client-truststore.p12 \
   $NEW_IS_HOME/repository/resources/security/
```

### Buoc 3: Chuyen custom JARs

```bash
# Sao chep custom OSGi bundles
cp $OLD_IS_HOME/repository/components/dropins/custom-*.jar \
   $NEW_IS_HOME/repository/components/dropins/

# Sao chep thu vien ben ngoai
cp $OLD_IS_HOME/repository/components/lib/custom-*.jar \
   $NEW_IS_HOME/repository/components/lib/
```

> **Luu y quan trong:** KHONG sao chep toan bo thu muc `dropins/` hoac `lib/`. Chi sao chep nhung JAR ma ban da them thu cong. Cac JAR he thong da duoc cap nhat trong ban moi.

### Buoc 4: Chuyen custom themes

```bash
# Neu da tuy chinh giao dien
cp -r $OLD_IS_HOME/repository/deployment/server/webapps/accountrecoveryendpoint/custom/ \
      $NEW_IS_HOME/repository/deployment/server/webapps/accountrecoveryendpoint/custom/
```

---

## 4. Cap nhat Database Schema

### 4.1 Tai migration client

Migration client nam tai repository rieng:
https://github.com/wso2-extensions/identity-migration-resources

```bash
# Tai migration client tuong ung voi phien ban 7.2.0
# Dat vao thu muc dropins
cp org.wso2.carbon.is.migration-*.jar \
   $NEW_IS_HOME/repository/components/dropins/
```

### 4.2 Chay migration scripts (vi du MySQL)

```sql
-- Kiem tra schema hien tai
SELECT * FROM IDN_IDENTITY_META_DATA WHERE PROPERTY_NAME = 'version';

-- Chay migration scripts theo thu tu
-- Scripts duoc cung cap trong migration client package
-- Vi du: Chay tu 7.0.0 -> 7.2.0
SOURCE migration-resources/7.0.0-to-7.2.0/dbscripts/step1/identity/mysql.sql;
SOURCE migration-resources/7.0.0-to-7.2.0/dbscripts/step1/um/mysql.sql;
```

### 4.3 Vi du cho PostgreSQL

```sql
-- Tuong tu cho PostgreSQL
\i migration-resources/7.0.0-to-7.2.0/dbscripts/step1/identity/postgresql.sql
\i migration-resources/7.0.0-to-7.2.0/dbscripts/step1/um/postgresql.sql
```

---

## 5. Cap nhat Configuration

### 5.1 Cac thay doi trong deployment.toml

So sanh va cap nhat `deployment.toml` giua 2 ban:

```toml
# === Cac cau hinh giu nguyen (khong can thay doi) ===

[server]
hostname = "your-hostname.com"    # Giu nguyen

[super_admin]
username = "admin"                # Giu nguyen
password = "your-password"        # Giu nguyen

[database.identity_db]            # Giu nguyen connection strings
type = "mysql"
url = "jdbc:mysql://localhost:3306/WSO2IDENTITY_DB"
username = "wso2carbon"
password = "wso2carbon"

[database.shared_db]              # Giu nguyen connection strings
type = "mysql"
url = "jdbc:mysql://localhost:3306/WSO2SHARED_DB"
username = "wso2carbon"
password = "wso2carbon"

[keystore.primary]                # Giu nguyen keystore config
file_name = "wso2carbon.p12"
password = "wso2carbon"
type = "PKCS12"

# === Cau hinh moi trong 7.2.0 (kiem tra va them neu can) ===

# Push notification (moi trong 7.2.0)
# [authentication.authenticator.push_notification]
# enable = true

# Visual login flow designer (tu dong bat)
# Khong can cau hinh them

# FAPI 2.0 support (neu can)
# [oauth.oidc.fapi]
# enabled = true
```

### 5.2 Cac config deprecated hoac thay doi

| Config cu (7.0.0) | Thay doi trong 7.2.0 | Hanh dong |
|-------|---------|---------|
| `[authentication.adaptive]` | Van ho tro, them options moi | Kiem tra options moi |
| Console UI config | Nhieu options moi cho visual designer | Review documentation |
| FAPI config | FAPI 2.0 support | Cap nhat neu dung FAPI |

### 5.3 Cap nhat log4j2

```bash
# So sanh log4j2 config
diff $OLD_IS_HOME/repository/conf/log4j2.properties \
     $NEW_IS_HOME/repository/conf/log4j2.properties

# Khuyen nghi: dung file log4j2.properties moi va chi them lai
# cac custom log appenders/loggers cua ban
```

---

## 6. Chay Migration Client

### 6.1 Cau hinh migration

Them vao `deployment.toml`:

```toml
[migration]
migrate = true
current_version = "7.0.0"
migrate_version = "7.2.0"
```

### 6.2 Khoi dong server voi migration

```bash
cd $NEW_IS_HOME/bin

# Chay server - migration client se tu dong chay
./wso2server.sh

# Theo doi log
tail -f $NEW_IS_HOME/repository/logs/wso2carbon.log | grep -i "migration"
```

### 6.3 Sau khi migration hoan tat

```toml
# Xoa hoac comment cau hinh migration trong deployment.toml
# [migration]
# migrate = true
```

Khoi dong lai server binh thuong.

---

## 7. Khoi dong va kiem tra

### 7.1 Khoi dong server

```bash
cd $NEW_IS_HOME/bin
./wso2server.sh
```

### 7.2 Kiem tra co ban

```bash
# 1. Kiem tra server da khoi dong thanh cong
curl -k https://localhost:9443/carbon/admin/login.jsp
# Ket qua: HTTP 200

# 2. Kiem tra Console UI
# Truy cap: https://localhost:9443/console
# Dang nhap: admin / <password>

# 3. Kiem tra MyAccount
# Truy cap: https://localhost:9443/myaccount

# 4. Kiem tra REST APIs
curl -k -u admin:admin https://localhost:9443/api/server/v1/configs
# Ket qua: JSON response voi server configs

# 5. Kiem tra SCIM2
curl -k -u admin:admin https://localhost:9443/scim2/Users
# Ket qua: List users

# 6. Kiem tra health
curl -k https://localhost:9443/api/health-check/v1.0/health
```

### 7.3 Kiem tra chuc nang

- [ ] Dang nhap Console voi admin account
- [ ] Tao va xoa user thu
- [ ] Dang ky ung dung OAuth2 moi
- [ ] Test luong SAML SSO (neu dung)
- [ ] Test luong OIDC (neu dung)
- [ ] Test adaptive authentication scripts
- [ ] Kiem tra custom authenticators hoat dong
- [ ] Test user self-registration (neu bat)
- [ ] Test password recovery
- [ ] Kiem tra audit logs

### 7.4 Kiem tra tinh nang moi 7.2.0

- [ ] Visual Login Flow Designer (Console > Applications > Login Flow)
- [ ] Push notification authentication (neu cau hinh)
- [ ] Cac improvements FAPI 2.0 (neu su dung)

---

## 8. Rollback plan

Neu nang cap that bai:

```bash
# 1. Dung server 7.2.0
cd $NEW_IS_HOME/bin
./wso2server.sh --stop

# 2. Restore database tu backup
mysql -u wso2carbon -p WSO2IDENTITY_DB < identity_db_backup.sql
mysql -u wso2carbon -p WSO2SHARED_DB < shared_db_backup.sql

# 3. Khoi dong lai server 7.0.0
cd $OLD_IS_HOME/bin
./wso2server.sh

# 4. Kiem tra server 7.0.0 hoat dong binh thuong
curl -k https://localhost:9443/carbon/admin/login.jsp
```

---

## Tong ket cac buoc

```
1. Backup toan bo (database + files + keystores)
2. Tai va giai nen IS 7.2.0
3. Chuyen deployment.toml, keystores, custom JARs
4. Chay database migration scripts
5. Cau hinh migration client trong deployment.toml
6. Khoi dong server (migration tu dong chay)
7. Xoa cau hinh migration
8. Kiem tra chuc nang
```

**Thoi gian uoc tinh:** 2-4 gio (tuy thuoc do phuc tap cua cau hinh hien tai)

**Tai lieu tham khao chinh thuc:**
- Migration Guide: https://is.docs.wso2.com/en/7.2.0/deploy/upgrade/upgrade-wso2-is/
- Migration Client: https://github.com/wso2-extensions/identity-migration-resources
- Release Notes: https://is.docs.wso2.com/en/7.2.0/references/release-notes/
