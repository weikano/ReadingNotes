### ATC（Augmented Traffic Control）

> ATC是Facebook开源的用于模拟不同网络环境的工具。它可以控制一个设备跟网络的连接。开发者可以用它来测试应用在不同网络环境下的情形，比如高速、移动甚至受损网络。可以控制的变量包括：带宽、延迟、丢包、损坏的包以及包的顺序。
>
> 要控制网络，ATC必须在网关之类的设备上运行。

#### 安装说明

```shell
pip install atc_thrift atcd django-atc-api django-atc-demo-ui django-atc-profile-storage
```

#### Django

新建一个Django项目

```shell
django-admin startproject actui
```

配置信息

```python
# atcui/actui/settings.py
INSTALLED_APPS = (
    # ...
    # Django ATC API
    'rest_framework',
    'atc_api',
    # Django ATC Demo UI
    'bootstrap_themes',
    'django_static_jquery',
    'atc_demo_ui',
    # Django ATC Profile Storage
    'atc_profile_storage',
)
```

然后配置actui/actui/urls.py

```python
# actui/actui/urls.py
# ...
from django.views.generic.base import RedirectView
from django.conf.urls import include

urlpatterns = [
    # ...
    # Django ATC API
    url(r'^api/v1/', include('atc_api.urls')),
    # Django ATC Demo UI
    url(r'^atc_demo_ui/', include('atc_demo_ui.urls')),
    # Django ATC profile storage
    url(r'^api/v1/profiles/', include('atc_profile_storage.urls')),
    url(r'^$', RedirectView.as_view(url='/atc_demo_ui/', permanent=False)),
]
```

最后更新Django DB

```shell
python actui/manage.py migrate
```

#### 运行ATC

