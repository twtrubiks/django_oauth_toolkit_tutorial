# django_oauth_toolkit_tutorial

本篇文章學習 OAuth2 的流程 (建議先 google 了解一下 OAuth2)

[OAuth Grant Types](https://oauth.net/2/grant-types/)

這篇範例會帶大家走一輪以下兩個流程

Authorization Code 以及 Client Credential

# 初始化

```cmd
pip install django-oauth-toolkit
```

文件參考 [django-oauth-toolkit](https://django-oauth-toolkit.readthedocs.io/en/latest/getting_started.html)

建立一個 users

```cmd
python manage.py startapp users
```

將 [users/models.py](users/models.py) 改成以下

```python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    pass
```

接著記得加入 INSTALLED_APPS,

```python
INSTALLED_APPS = [
    ......
    'users.apps.UsersConfig',
    ......
]
```

再設定

```python
AUTH_USER_MODEL='users.User'
```

migrate

```cmd
python manage.py makemigrations
python manage.py migrate
```

接著開始 Django OAuth Toolkit

先在 `settings.py` INSTALLED_APPS 底下加入 oauth2_provider

```python
INSTALLED_APPS = [
    ......
    'users.apps.UsersConfig',
    'oauth2_provider',
]
```

在 `urls.py` 中加入

```python
path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
```

migrate

```cmd
python manage.py migrate
```

最後在 `settings.py` 設定

`LOGIN_URL='/admin/login/'` 直接使用 admin登入, 讓整個流程簡單一點

建立 superuser

```cmd
python manage.py createsuperuser
```

把 django run 起來

```cmd
python manage.py runserver
```

# Authorization Code

先使用最完整的 Authorization code

到以下網址 建立 Client id 和 Client secret

Authorization Grant Type 選擇 Authorization code

[http://127.0.0.1:8000/o/applications/register/](http://127.0.0.1:8000/o/applications/register/)

![alt tag](https://i.imgur.com/PrJVLBB.png)

Redirect uris 網址先隨便填

```text
http://127.0.0.1:8000/noexist/callback
```

注意, Client secret 保存前就要先複製, 因為如果 save 後他會是 hashed 過的.

接著進行 Proof Key for Code Exchange

```python
import random
import string
import base64
import hashlib

code_verifier = ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(random.randint(43, 128)))

code_challenge = hashlib.sha256(code_verifier.encode('utf-8')).digest()
code_challenge = base64.urlsafe_b64encode(code_challenge).decode('utf-8').replace('=', '')
```

假設這邊的 `code_verifier`

```text
UATR8W69WOXAJLC1BSJXW9QTT2KA6HOWH17AAAP9JVUR4URQDD16SB0F0S7VD8B6DR8Z6L7M8IB2OINO7A6TQ90
```

`code_challenge`

```text
ZUbNoUIiLIyk_f3-nHWyFVRoSzyjFhk9zUSREV__eDY
```

進入以下網址

```text
http://127.0.0.1:8000/o/authorize/?response_type=code&code_challenge=ZUbNoUIiLIyk_f3-nHWyFVRoSzyjFhk9zUSREV__eDY&code_challenge_method=S256&client_id=3ljDU5ninYH79sRYPNSk81QMl8CkS7gVd7erXcvu&redirect_uri=http://127.0.0.1:8000/noexist/callback
```

請將 `code_challenge` 以及 `client_id` 修改成自己的,

進入頁面後, 按下 Authorize,

![alt tag](https://i.imgur.com/fKK7Skf.png)

你會得到類似的 (網址是 404 是正常的, 因為網址本來就不存在)

![alt tag](https://i.imgur.com/KWpnERt.png)

```text
http://127.0.0.1:8000/noexist/callback?code=LDzrgTJGY7jdj9ynmOEZbYLQHHWMkw
```

`code` 是 `LDzrgTJGY7jdj9ynmOEZbYLQHHWMkw`

接著可以使用這個 code 去取得你的 token,

```cmd
curl -X POST \
    -H "Cache-Control: no-cache" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    "http://127.0.0.1:8000/o/token/" \
    -d "client_id=${ID}" \
    -d "client_secret=${SECRET}" \
    -d "code=${CODE}" \
    -d "code_verifier=${CODE_VERIFIER}" \
    -d "redirect_uri=http://127.0.0.1:8000/noexist/callback" \
    -d "grant_type=authorization_code"
```

`${ID}` 放入 client_id

`${SECRET}` 放入 client_secret

`${CODE}` 放入 code

`${CODE_VERIFIER}` 放入 code_verifier

提供一個用 python 的版本,

```python
import requests

url = "http://127.0.0.1:8000/o/token/"
payload = {
    "client_id": "client_id",
    "client_secret": "client_secret",
    "code": "code",
    "code_verifier": "code_verifier",
    "redirect_uri": "http://127.0.0.1:8000/noexist/callback",
    "grant_type": "authorization_code"
}
headers = {
    "Cache-Control": "no-cache",
    "Content-Type": "application/x-www-form-urlencoded"
}

response = requests.post(url, data=payload, headers=headers)
print(response.text)
```

你應該會得到類似的 response, 成功得到 access_token,

注意, 這個 code 只能用一次,

當你使用了, 這個 code 就不會在資料庫的 `oauth2_provider_grant` 裡面了.

```json
{
    "access_token": "AIqCg9FckVo8d67cCdXhs43fjFuodD",
    "expires_in": 36000,
    "token_type": "Bearer",
    "scope": "read write",
    "refresh_token": "k9oJHrvL3pvUpskaMOdF8RpZokBST2"
}
```

最後可以從這個 token 去拿資源

```cmd
curl \
    -H "Authorization: Bearer jooqrnOrNa0BrNWlg68u9sl6SkdFZg" \
    -X GET http://localhost:8000/resource
```

這網址因為我們沒定義, 所以暫時不存在是正常的.

# Client Credential

適合使用在 machine-to-machine authentication,

只用 Client id 和 Client secret 就可以拿到 token,

流程更簡單一點,

到以下網址 建立 Client id 和 Client secret,

Authorization Grant Type 選擇 Client credential,

[http://127.0.0.1:8000/o/applications/register/](http://127.0.0.1:8000/o/applications/register/)

![alt tag](https://i.imgur.com/ysso2v7.png)

接下來 encode client_id 和 client_secret

範例如下

```python
>>> import base64
>>> client_id = "{client_id}"
>>> secret = "{secret}"
>>> credential = "{0}:{1}".format(client_id, secret)
>>> base64.b64encode(credential.encode("utf-8"))
b'${CREDENTIAL}'
```

著接把 ${CREDENTIAL} 帶進去

```cmd
curl -X POST \
    -H "Authorization: Basic ${CREDENTIAL}" \
    -H "Cache-Control: no-cache" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    "http://127.0.0.1:8000/o/token/" \
    -d "grant_type=client_credentials"
```

你會得到類似的結果

```json
{
    "access_token": "7xFTE6fuPdMdKqy9Ho0gSmcLxk6qbS",
    "expires_in": 36000,
    "token_type": "Bearer",
    "scope": "read write"
}
```


## 執行環境

* Python 3.9
* Django 4.2.6

## Reference

* [Django OAuth Toolkit Documentation](https://django-oauth-toolkit.readthedocs.io/en/latest/)
* [OAuth Grant Types](https://oauth.net/2/grant-types/)

## Donation

文章都是我自己研究內化後原創，如果有幫助到您，也想鼓勵我的話，歡迎請我喝一杯咖啡:laughing:

![alt tag](https://i.imgur.com/LRct9xa.png)

[贊助者付款](https://payment.opay.tw/Broadcaster/Donate/9E47FDEF85ABE383A0F5FC6A218606F8)

## License

MIT license