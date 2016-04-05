# Keystone 安裝與設定
本章節會說明與操作如何安裝身份認證服務到 Controller 節點上，並設置相關參數與設定。若對於 Keystone 不瞭解的人，可以參考 [Keystone 身份認證服務章節](../../../conceptions/keystone/README.md)。

- [Keystone 安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [設定 Apache HTTP 伺服器](#設定-apache-http-伺服器)
- [建立 Service 與 API Endpoint](#建立-service-與-api-endpoint)
- [建立 Keystone admin 與 user](#建立-keystone-admin-與-user)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Keystone 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Keystone 資料庫：
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
> 這邊```KEYSTONE_DBPASS```可以隨需求修改。

完成後離開資料庫，之後建立一個亂數 Token，使用以下指令：
```sh
$ openssl rand -hex 24
21d7fb48086e09f30d40be5a5e95a7196f2052b2cae6b491
```
> 該 Token 將提供給 Keystone 的 ```admin_token``` 使用。

### 套件安裝與設定
由於 Eventlet 將在後續版本被棄用，所以在安裝 Keystone 之前要手動設定服務在安裝完與開機時不自動啟動：
```sh
$ echo "manual" | sudo tee /etc/init/keystone.override
manual
```
> 在 Kilo 與 Liberty 版本中， Keystone 專案棄用了 Eventlet Networking，因此本教學將使用 Apache2 mod_wsgi 來提供身份證認服務，這邊預設下將佔用到 5000 與 35357 Port。因此在安裝 Keystone 時將停用原生服務。而在 Mitaka 版本將正式移除 Eventlet。

確認關閉自動啟動 Keystone 後，就可以直接安裝套件，透過以下指令安裝：
```sh
$ sudo apt-get install keystone apache2 memcached \
libapache2-mod-wsgi python-openstackclient python-memcache
```

安裝完後，編輯```/etc/keystone/keystone.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
admin_token = ADMIN_TOKEN
```
> 其中 ```ADMIN_TOKEN``` 取代成上面步驟建立的亂數 Token。

在```[database]```部分修改使用以下方式：
```sh
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@10.0.0.11/keystone
```

在```[memcache]```部分加入以下內容：
```
[memcache]
servers = 10.0.0.11:11211
```

在```[token]```部分加入以下內容：
```
[token]
provider = uuid
driver = memcache
```

在```[revoke]```部分加入以下內容：
```[revoke]
driver = sql
```

完成後，透過 Keystone 管理指令來同步資料庫建立資料表：
```sh
$ sudo keystone-manage db_sync
```

### 設定 Apache HTTP 伺服器
由於本教學採用 WSGI Mod 來提供 Keystone 服務，因此我們將使用到 Apache2 來建立 HTTP 服務，首先編```/etc/apache2/apache2.conf``` 加入以下內容：
```
ServerName 10.0.0.11
```

接著建立一個設定檔```/etc/apache2/sites-available/wsgi-keystone.conf```來提供 Keystone 服務，並加入以下內容：
```
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
```

確認檔案無誤後，就可以讓 Apache2 使用這個設定檔，透過以下指令：
```sh
$ sudo ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

完成後就可以重新啟動 Apache2 伺服器：
```sh
$ sudo service apache2 restart
```

最後由於第一次安裝時，預設會建立一個 SQLite 資料庫，因此這邊可以將之刪除：
```sh
$ sudo rm -f /var/lib/keystone/keystone.db
```

### 建立 Service 與 API Endpoint
在建立 Keystone service 與 Endpoint 之前，要先導入一些環境變數，由於 OpenStack client 會自動抓取系統某些環境變數來提供 API 的存取，首先透過以下指令導入：
```sh
export OS_TOKEN=21d7fb48086e09f30d40be5a5e95a7196f2052b2cae6b491
export OS_URL=http://10.0.0.11:35357/v2.0
```
> 其中```OS_TOKEN```是 Keystone 的 Admin Token。

完後上述後，就可以建立 Service 實體來提供身份認證：
```sh
$ openstack service create --name keystone \
--description "OpenStack Identity" identity
```

成功的話，會看到類似以下結果：
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 4ddaae90388b4ebc9d252ec2252d8d10 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```

Keystone 為了讓指定的 API 與 Service 擁有認證機制，故要再新增 API Endpoint 目錄給 Keystone，這樣就能夠決定如何與其他服務進行存取，透過以下方式建立：
```sh
$ openstack endpoint create \
--publicurl http://10.0.0.11:5000/v2.0 \
--internalurl http://10.0.0.11:5000/v2.0 \
--adminurl http://10.0.0.11:35357/v2.0 \
--region RegionOne identity
```
> P.S. 以上方式是採用 Keystone v2.0，若採用 v3 則需要更換其他方式。

成功的話，會看到類似以下結果：
```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://10.0.0.11:35357/v2.0      |
| id           | 0af1316372844e579bc84cf71f66fff0 |
| internalurl  | http://10.0.0.11:5000/vv2.0      |
| publicurl    | http://10.0.0.11:5000/v2.0       |
| region       | RegionOne                        |
| service_id   | 721282b3c5514bcab6147a29fbaf28e7 |
| service_name | keystone                         |
| service_type | identity                         |
+--------------+----------------------------------+
```

因此後續安裝的 OpenStack 各套件服務都需要建立一個或多個 Service，以及 API Endpoint 目錄。

### 建立 Keystone admin 與 user
身份認證服務會透過 Domains、Projects、Roles 與 Users 的組合來進行授權。在大多數部署下都會擁有管理者角色，因此這邊透過 OpenStack client 建立一個名稱為 ```admin``` 的管理者，以及一個專門給所有 OpenStack 套件溝通的```Service project```：
```sh
# 建立 admin Project
$ openstack project create --description "Administrator project" admin

# 建立 admin User
$ openstack user create --password passwd --email admin@example.com admin

# 建立 admin Role
$ openstack role create admin

# 加入 Project 與 User 到 admin Role
$ openstack role add --project admin --user admin admin

# 建立 service Project
$ openstack project create --description "Services project" service
```

接著建立一個一般使用者的 Project，來提供後續的權限驗證：
```sh
openstack project create --description "Demo Project" demo
openstack user create --password demo --email demo@example.com demo
openstack role create user
openstack role add --project demo --user demo user
```
> 以上指令可以重複的建立不同的 Project 與 User。

### 驗證服務
在進行其他服務安裝之前，一定要確認 Keystone 服務沒有任何錯誤，首先取消上面導入的環境變數：
```sh
unset OS_TOKEN OS_URL
```

接著透過 ```admin``` 使用者來驗證 v2.0 API 是否能正常取得一個 Token，透過 OpenStack client 來取得：
```sh
$ openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
token issue
```
> 本教學範例使用 ```passwd```。

成功的話，會看到類似以下結果：
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-07-23T15:36:06Z             |
| id         | 995e854225fc45efac11e3bdc705a675 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```

完成 v2.0 驗證後，要繼續進行 v3 的版本驗證，由於 v3 增加了 Domains 的驗證。因此 Project 與 User 能夠在不同的 Domain 使用相同名稱，這邊是使用預設的 Domain 進行驗證：
```sh
$ openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
token issue
```
> 其中 ```default``` 是當沒有指定 Domain 時的預設名稱。

成功的話，會看到類似以下結果：
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-07-23T15:36:06Z             |
| id         | 67ef08f69381404a8ccaf2a4725b58d5 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```

然後接下來要驗證權限是否正常被設定，這邊先使用 ```admin``` 使用者來進行：
```sh
$ openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
project list
```

成功的話，會看到類似以下結果：
```
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 675b421d5f794e10b924bffcfba6b3ab | demo    |
| b1214dcf4156464d8c6df2c20fd51e73 | service |
| cf8f9b8b009b429488049bb2332dc311 | admin   |
+----------------------------------+---------+
```

然後再透 ```demo``` 使用者來驗證是否有存取權限，這邊利用 v3 來取得 Token：
```sh
$ openstack --os-auth-url http://10.0.0.11:5000 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name demo \
--os-username demo \
--os-auth-type password \
token issue
```
> 本教學範例使用 ```demo```。

成功的話，會看到類似以下結果：
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-07-23T15:36:06Z             |
| id         | b1214dcf4156464d8c6df2c20fd51e73 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```

> P.S 這邊會發現使用的 Port 從 35357 轉換成 5000，這邊只是為了區別 Admin URL 與 Public URL 中使用的 Port。

最後再透過 ```demo``` 來使用擁有管理者權限的 API：
```sh
$ openstack --os-auth-url http://10.0.0.11:5000 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name demo \
--os-username demo \
--os-auth-type password \
user list
```

成功的話，會看到類似以下結果：
```
ERROR: openstack You are not authorized to perform the requested action: admin_required
```

若上述過程都沒有錯誤，表示 Keystone 目前很正常的被執行中。

### 使用腳本切換使用者
由於後續安裝可能會切換不同使用者來驗證一些服務，因此可以透過建立腳本來導入相關的環境變數，來達到不同使用者的切換，首先建立以下兩個檔案：
```sh
$ touch admin-openrc demo-openrc
```

編輯 ```admin-openrc``` 加入以下內容：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=passwd
export OS_AUTH_URL=http://10.0.0.11:35357/v3
export OS_IDENTITY_API_VERSION=3
```

編輯 ```demo-openrc``` 加入以下內容：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://10.0.0.11:5000/v3
export OS_IDENTITY_API_VERSION=3
```

完成後，可以透過 Linux 指令來執行檔案導入環境變數，如以下指令：
```sh
$ source admin-openrc
```
> 也可以使用以下方式來執行檔案：
```sh
$ . admin-openrc
```

完成後，在使用 OpenStack client 就可以省略一些基本參數了，如以下指令：
```sh
$ openstack token issue
```

成功的話，會看到類似以下結果：
```sh
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-07-23T15:36:06Z             |
| id         | b1214dcf4156464d8c6df2c20fd51e73 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```
