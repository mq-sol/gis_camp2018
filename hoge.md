#  django インストール手順
### アカウント情報
* ROOT
* djangodev

管理者アカウント
-    ID: root
-    PW: c3dj@ngo

ユーザーアカウント
-    ID: djangodev
-    PW: dj@ng0


### IPアドレス
192.168.99.105


## OSの設定
* セキュリティアップデート
```
# yum -y update
# yum -y upgrade
```
* Firewallに変更したポート追加
```
# firewall-cmd --add-port=10022/tcp --zone=public --permanent
# firewall-cmd --reload
# systemctl restart sshd
```
* selinuxは無効にする
```
$ sudo vi /etc/selinux/config
```
```
SELINUX=disabled
```

#### 参考URL[https://narito.ninja/detail/21/]
## Nginxのインストール
* repositoryの追加  
下記を/etc/yum.repo.d/ngix.repoに記載
```
$ sudo vi /etc/yum.repos.d/ngix.repo
```
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

* yumでインストール
```
$ sudo yum -y install nginx
```

* 自動起動設定
```
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --reload
$ sudo systemctl start nginx
$ sudo systemctl enable nginx
```

* スタティックディレクトリ作成
```
$ sudo mkdir /usr/share/nginx/html/media
$ sudo mkdir /usr/share/nginx/html/static
```
* project.confの作成
```
$ sudo vi /etc/nginx/conf.d/project.conf
```
```
server {
    listen  80;
    server_name 192.168.99.105;

    location /static {
        alias /usr/share/nginx/html/static;
    }

    location /media {
        alias /usr/share/nginx/html/media;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```



## PostgreSQLのインストール
* リポジトリの追加
```
$ sudo yum -y localinstall https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
```
* インストール
```
$ sudo yum -y groupinstall "PostgreSQL Database Server 11 PGDG"
```

* 初期化
```
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
```

* 起動
```
$ sudo systemctl start postgresql-11
$ sudo systemctl enable postgresql-11
```

* DBの作成(testdb)、ユーザー(dbuser)の作成、
```
$ sudo -u postgres createdb -E UTF-8 --locale ja_JP.UTF-8 -T template0 testdb
$ sudo -u postgres psql
```
```
postgres=# CREATE USER dbuser WITH PASSWORD 'dbpass';
postgres=# ALTER ROLE dbuser SET client_encoding TO 'utf8';
ALTER ROLE dbuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE dbuser SET timezone TO 'Asia/Tokyo';ALTER ROLE
postgres=# ALTER ROLE dbuser SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE dbuser SET timezone TO 'Asia/Tokyo';
postgres=# GRANT ALL PRIVILEGES ON DATABASE testdb TO dbuser;
```
* アクセス権限の修正  
 identをmd5に変更
```
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
* DB再起動
```
$ sudo systemctl restart postgresql-11
```

## Djangoのインストール
### Python3のインストール
* yumリポジトリ追加
* python3のインストール
```
$ sudo yum install -y https://centos7.iuscommunity.org/ius-release.rpm
$ sudo yum install -y python36u python36u-libs python36u-devel python36u-pip
```
### Postgresqlドライバのインストール
```
$ sudo pip3.6 install psycopg2
$ sudo pip3.6 install psycopg2-binary
```

### djangoのインストール
```
$ sudo pip3.6 install django
```
* プロジェクトの作成
```
$ django-admin.py startproject mysite
$ python3.6 manage.py startapp myapp
```
* 設定ファイルの編集  
下記を修正
```
vi mysite/mysite/settings.py
```
```   
# 修正部分
ALLOWED_HOSTS = ["*"]
LANGUAGE_CODE = 'ja'

TIME_ZONE = 'Asia/Tokyo'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'testdb',
        'USER': 'dbuser',
        'PASSWORD': 'dbpass',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

# 追記部分
STATIC_ROOT = '/usr/share/nginx/html/static'
MEDIA_URL = '/media/'
MEDIA_ROOT = '/usr/share/nginx/html/media'
```
* スタティックファイルのコピー
```
$ sudo python3.7 manage.py collectstatic
```

* マイグレーションとか
```
$ python3.6 manage.py makemigrations
$ python3.6 manage.py migrate
```
* スーパーユーザー作成  
ID: webadmin  
PW: webpassword  
```
$ python3.6 manage.py createsuperuser
```
```
メールアドレス: webadmin@mq-sol.jp
Password:
Password (again):
Superuser created successfully.
```

* 起動及びファイアウォール設定
```
$ python3.6 mysite/manage.py runserver 127.0.0.1:8000
$ sudo firewall-cmd --add-port 8000/tcp --permanent
$ sudo firewall-cmd --reload
```

* ブラウザで確認  
[http://192.168.99.105:8080]

### Gunicornのインストール
```
$ sudo pip3.6 install gunicorn
```
* デーモンとして起動
```
$ cd mysite
$ sudo gunicorn --daemon --bind 127.0.0.1:8000 mysite.wsgi:application
```



