---
title: mac下使用脚本为图片加水印、上传图床
date: 2018-06-13
categories: Utils
tags:
- mac
- utils
---
## 为什么会有这样的需求？

笔者经常用markdown写一些东西，图片是文章中必不可少的。一般情况下，都是先是将画好的图保存到本地的某个地方，然后手动将图片上传到OSS（如七牛云），最后拷贝外链地址加到文章中。
 另一种场景是加水印。原创不易，有时候会发生第一天晚上写好的博客文章，第二天就出现在某公众号号，并且申请了原创保护。这个时候如果图片上有个特殊水印，原封不动地拷贝文章就有出处可寻。虽然不能完全解决这种侵权的行为，但是也提高了抄袭者的成本。


## 具体实现

至于具体实现，我当时考虑了几个问题：

1. 文件下载了之后，能不能立即将该文件夹的变化输出；
2. 将下载后的文件能够加上预置好的水印（这个OSS肯定能够提供）；
3. 根据输出的新增文件名（图片名），将该文件上传到OSS；
4. 返回图片文件的外链地址。


涉及到的工具：

- OS: macOS
- 图床：七牛云
- 文件监控工具：fswatch

 
笔者整理了使用的七牛云的存储SDK，主要是`上传&预转持续化`，将图片加水印上传到对应的bucket中。这仅是一个小工具，目前已经能满足笔者的需求。

### 文件监控工具：fswatch
Linux下面可以使用[inotify-tools](http://linux.die.net/man/1/inotifywatch )来进行文件夹、文件变更的检测。

fswatch是一个使用Mac OS X FSEvents API的同步工具，同时也可以使用在BSD 与Debian操作系统。

使用Homebrew进行安装：

```bash
brew install fswatch
```
监控/User/usr/Downloads文件夹：

```bash
fswatch /User/usr/Downloads
```
当/User/usr/Downloads文件内容变化时，输出变动的文件列表。如：

```
/Users/user/Downloads/PlatformTransactionManager.jpg
```
当然fswatch还有其他用法，这里不展开。
### 七牛云创建bucket以及样式
 首先需要注册七牛云的账号，点击[注册通道](https://portal.qiniu.com/signup?code=3lffq8tzqxc9e)注册（笔者这里不是专门打广告，没有收取任何广告费。。）。
 
其次，创建一个bucket。如下图，笔者命名为`bl-bucket`。
 
![](http://image.blueskykong.com/bucket-bl.jpg)

在上图中，还可以看出笔者绑定了域名`http://image.blueskykong.com`，这个可以自行选择，如果不绑定，可以使用测试域名。绑定域名之后，记得将该域名设为外链默认域名。

![](http://image.blueskykong.com/default-bucket-name.jpg)

这里之所以这样做，是为了后面脚本返回最后的外链地址（我定义的规则为：拼接外链默认域名 + 文件名）。

然后，设置水印样式，笔者在图片的右下角添加了文字水印`公众号：aoho求索`。当然还可以对图片进行其他样式设置，如缩放图片，设置长宽等，读者根据需要自定。设置好之后，如下图所示：

![](http://image.blueskykong.com/css-bucket.jpg)

我们需要的其实是，最后生成的处理接口，这会用在我们的上传脚本中。
### 脚本编写

上面的步骤中，我们先是记录了特定文件的新增图片名记录，其次是设定好我们的bucket、外链域名以及水印样式。下面我们进行编码。

启动fswatch后，输出变动文件记录到指定的文件，之后主要分为三步：

- 我们的脚本基于Python，安装七牛云的Python SDK，具体参考官网[Python SDK](https://developer.qiniu.com/kodo/sdk/1242/python#3)
- 读取指定文件记录的变化的值，这里笔者指定读取最后一行（变动其实是主动的，所以只需要最后一行的记录即为新增的图片文件）
- 将该文件进行上传，用到的是七牛云的`上传&预转持续化`。

```python
# -*- coding: utf-8 -*-
# flake8: noqa

from qiniu import Auth, put_file, etag, urlsafe_base64_encode
import qiniu.config
from qiniu import BucketManager
from optparse import OptionParser
# 水印上传函数
def watermark_add(filename):
	#要上传的空间
	bucket_name = 'bl-bucket'

	#上传到七牛后保存的文件名
	arr = filename.split("/")
	
	key = arr[len(arr)-1]
	# 设置图片缩略参数
	fops = 'imageView2/0/q/75|watermark/2/text/5YWs5LyX5Y-377yaYW9ob-axgue0og==/font/5b6u6L2v6ZuF6buR/fontsize/400/fill/IzAwMDAwMA==/dissolve/50/gravity/SouthEast/dx/10/dy/10'

	# 通过添加'|saveas'参数，指定处理后的文件保存的bucket和key，不指定默认保存在当前空间，bucket_saved为目标bucket，key_saved为目标key
	encode_str = bucket_name + ":" + key

	saveas_key = urlsafe_base64_encode(encode_str)

	fops = fops+'|saveas/'+saveas_key

	access_key = 'xxx'
	secret_key = 'xxx'

	#构建鉴权对象
	q = Auth(access_key, secret_key)

	#生成上传 Token，可以指定过期时间等
	# 在上传策略中指定fobs和pipeline
	policy={
	  'persistentOps':fops
	 }

	token = q.upload_token(bucket_name, key, 3600, policy)

	#要上传文件的本地路径
	localfile = filename

	ret, info = put_file(token, key, localfile)
	print("http://image.blueskykong.com/" + key)
	assert ret['key'] == key
	assert ret['hash'] == etag(localfile)
	pass

# 直接上传函数
def upload_pic(filename):
	....#较为简单，此处省略

# 获取变化的文件名
def get_upload_name(fname):
  with open(fname, 'r') as f:  #打开文件
    lines = f.readlines() #读取所有行
    first_line = lines[0] #取第一行
    last_line = lines[-1] #取最后一行
    return last_line

fname='nohup.out'
if __name__ == '__main__':
  l = get_upload_name(fname)
  l=l.replace("\n", "")
#  print (l)
  watermark_add(l)
#  print (upload_pic(l))
```
如上的脚本即可，读者需要自行设置access_key、secret_key、fname（fswatch输出的文件名）、bucket_name。秘钥在个人中心可以找到：

![](http://image.blueskykong.com/secret-qiniu.jpg)

### 测试

启动fswatch监听/Users/user/Documents/pic/，并输出记录到nohup.out文件中。

```bash
nohup fswatch /Users/user/Documents/pic/ &
```

fswatch会在后台进程持续监听。笔者还写了个bash命令，脚本可以写得很简单，如下：

```bash
#! /bin/bash
echo "uploading pic..."
echo "url is : "
python upload.py
exit 0
```
每次上传图片后，执行该命令直接得到到如下的输出：

```
uploading pic...
url is :
http://image.blueskykong.com/secret-qiniu.jpg
```
这是在本地截图保存之后，执行脚本的输出内容。`http://image.blueskykong.com/secret-qiniu.jpg`即为我们上传图片的外链地址。

## 总结
一个小工具的分享，适合在mac下用markdown写作的小伙伴，减轻一些繁琐的操作。目前来说，能满足笔者的需求，也没啥高端的地方。如果大家有改进或者更好的工具，记得留言分享。当然，如果有不清楚的地方，可以参考git项目https://github.com/keets2012/Sync-Pic.git。

七牛云注册地址： https://portal.qiniu.com/signup?code=3lffq8tzqxc9e
#### 参考
[OS X使用fswatch+rsync自动检测文件夹改动并同步](https://my.oschina.net/mengshuai/blog/618354)

