---
title: 'Django: 头像数据库'
date: 2022-11-03 14:59:32
tags: Web
---

## 引言

Django 默认将文件（`models.FileField`）、图片（`models.ImageField`）存储到文件系统中，仅在数据库中存储文件的相对路径，这种方式不适合存储用户头像：32x32 的 PNG 格式的头像的大小一般在 2KB 左右，从大小上看仅相当于一个`models.TextField`，而且在在数据库中更便于查询和管理。因此，在最近的项目中（预计不久后会上传到 [Github](https://github.com/vhqr0/zzuexp)）我尝试用数据库存储用户头像。

## 模型定义

models.py:
```python
from django.db import models
from django.contrib.auth.models import User

from io import BytesIO
from PIL import Image

class Avatar(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    data = models.BinaryField(max_length=4096)

    @classmethod
    def set(cls, user, img):
        bio = BytesIO()
        img = Image.open(img).resize((32, 32))
        img.save(bio, format='png')
        Avatar.objects.update_or_create(user=user,
                                        defaults={'data': bio.getvalue()})
```

模型最好的实现方式是在 `User` 中添加一个外键到 `Avatar`，然而 `User` 已经被写死了，只能 `Avatar` 引用 `User`。前者可以在模板生成阶段获取到 `Avatar` 的主键进而生成图片链接或者是默认的静态图片链接。而后者则仅能用 `User` 的主键生成图片链接，然后后端反向查询 `Avatar` 然后返回或重定向到静态图片。

`OneToOneField(User)` 基本上等价于 `ForeignKey(User, unique=True)`，但是 `User` 的方向链接的格式发生了改变。后者反向链接的格式是查询集 `user.avatar_set`，可以为空。而前者的则默认 `User` 和 `Avatar` 是双射，反向链接的格式是模型对象 `user.avatar`，然而双射无论是在数据库层面还是框架代码层面都无法保证，即可能有的 `User` 没有对应的 `Avatar`，此时反向引用将抛出未知的错误。我在项目中使用如下方法获取：

```python
if hasattr(user, 'avatar):
    avatar = user.avatar
    # get avatar
else:
    # default avatar
```

类方法 `Avatar.set` 展示了如何处理图片数据，`img` 是表单处理后得到的 `forms.ImageField` 类型对象，可以用 `PIL.Image` 打开，然后将图片压缩映射到 32x32 后存入数据库。在此处定义类方法避免表单层和视图层操作第三方库修改数据库和引用其他模型。

## 上传表单定义

forms.py:
```python
from django import forms
from .models import Avatar

class AvatarUploadForm(forms.Form):
    avatar = forms.ImageField(help_text='Please upload an image < 4KB.')

    def clean(self):
        if self.cleaned_data['avatar'].size > 4096:
            raise forms.ValidationError('Image too big!')
        return super().clean()

    def save(self, request):
        Avatar.set(request.user, self.cleaned_data['avatar'])
```

重写 `clean` 方法可以在 `form.is_valid()` 的收尾阶段执行额外的检查，这里检查上传图片的大小不能超过 4KB。`save` 在表单层中进一步封装模型层的方法以避免视图层直接操作模型层。

## 上传处理视图

views.py
```python
from django.views.generic import FormView
from .forms import AvatarUploadForm

class GenericFormTemplateMixin:
    # ...
    pass

class AvatarUploadView(GenericFormTemplateMixin, FormView):
    form_class = AvatarUploadForm
    legend = 'Avatar Upload'
    enctype = 'multipart/form-data'

    def form_valid(self, form):
        form.save(self.request)
        return super().form_valid(form)

    def get_success_url(self):
        return reverse('indexer:user-detail', args=(self.request.user.pk, ))
```

一个简单的 `FormView`，值得注意的是前端表单上传的 `enctype` 必须设置为 `multipart/form-data`。文件数据存储在 `request.FILES` 里，如果不使用通用视图，应将其传递给 `form_class`, 例如：`form = AvatarUploadForm(request.POST, request.FILES)`。

## 头像获取视图

views.py
```python
from django.shortcuts import get_object_or_404
from django.templatetags.static import static
from django.http import HttpResponse, HttpResponseRedirect
from django.contrib.auth.models import User
from .models import VerifyRecord, UserProfile, Category, Page

@require_GET
def avatar(request, pk):
    user = get_object_or_404(User, pk=pk)
    if hasattr(user, 'avatar'):
        return HttpResponse(user.avatar.data,
                            headers={'Content-Type': 'image/png'})
    else:
        return HttpResponseRedirect(
            static('indexer/images/default_avatar.png'))
```

如果找到头像，则将头像数据传给客户端，`Content-Type` 为 `image/png`，否则重定向到默认头像的静态图片地址。
