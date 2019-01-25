# alias导致venv异常的分析和解法
python3中venv模块可以虚拟出一个独立的Python环境，在这个环境中安装的第三方库不会对系统中的Python产生影响。Python3.3以上的版本通过venv模块原生支持虚拟环境，可以代替Python之前的virtualenv。该venv模块提供了创建轻量级“虚拟环境”，提供与系统Python的隔离支持。
## Python的运行方式
一般我们会使用以下两种方式之一来运行Python：  
```
python xxx.py
```  
或者在代码的第一行加上python的路径：  
```
#! /usr/local/bin/python
```  
这两种方式，使用的是系统中的python来解释代码。
## 问题复现
如果电脑上安装了Python2和Python3，那么想运行Python3写的代码的时候，我们可以使用以下方法来运行：  
```
python3 xxx.py
```  
但是由于不想写数字3，于是就使用了zsh的alias功能，在/etc/profile.d路径下新加文件python.sh，内容为：  
```
alias python='/usr/local/bin/python3.5'
```  
在这种情况下，使用：  
```
python xxx.py
```  
就可以通过Python3来解析代码了。这种方式使用系统中的Python没有问题，但是在虚拟环境venv下就出问题了。
我们创建一个虚拟环境test\_venv并激活，安装Python的Django库，并创建项目test\_django，代码流程如下：  
```
[root@localhost Desktop]# mkdir test_django
[root@localhost Desktop]# cd test_django/
[root@localhost test_django]# python -m venv test_venv
[root@localhost test_django]# source test_venv/bin/activate
(test_venv) [root@localhost test_django]# pip install Django
Collecting Django
  Using cached https://files.pythonhosted.org/packages/36/50/078a42b4e9bedb94efd3e0278c0eb71650ed9672cdc91bd5542953bec17f/Django-2.1.5-py3-none-any.whl
Collecting pytz (from Django)
  Using cached https://files.pythonhosted.org/packages/61/28/1d3920e4d1d50b19bc5d24398a7cd85cc7b9a75a490570d5a30c57622d34/pytz-2018.9-py2.py3-none-any.whl
Installing collected packages: pytz, Django
Successfully installed Django-2.1.5 pytz-2018.9
(test_venv) [root@localhost test_django]# django-admin.py startproject test_django .
(test_venv) [root@localhost test_django]# ll
total 4
-rwxr-xr-x. 1 root root 543 Jan 24 22:22 manage.py
drwxr-xr-x. 2 root root  74 Jan 24 22:22 test_django
drwxr-xr-x. 5 root root 100 Jan 24 22:22 test_venv
```  
至此，我们已经成功创建了test\_django项目，下面我们需要创建数据库：  
```
(test_venv) [root@localhost test_django]# python manage.py migrate
Traceback (most recent call last):
  File "manage.py", line 8, in <module>
    from django.core.management import execute_from_command_line
ImportError: No module named 'django'
The above exception was the direct cause of the following exception:
Traceback (most recent call last):
  File "manage.py", line 14, in <module>
    ) from exc
ImportError: Couldn't import Django. Are you sure it's installed and available on your PYTHONPATH environment variable? Did you forget to activate a virtual environment?
``` 
报错No module named 'django',用python命令行导入还是一样的报错：    
```
(test_venv) [root@localhost test_django]# python
Python 3.5.0 (default, Dec 28 2018, 00:17:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-36)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import django
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named 'django'
```
于是打开test_venv/lib/python3.5/site-packages却发现django安安静静的躺在里面。
```
(test_venv) [root@localhost test_django]# cd test_venv/lib/python3.5/site-packages/
(test_venv) [root@localhost site-packages]# ll
total 16
drwxr-xr-x. 19 root root 4096 Jan 24 22:22 django
drwxr-xr-x.  2 root root  109 Jan 24 22:22 Django-2.1.5.dist-info
-rw-r--r--.  1 root root  126 Jan 24 22:21 easy_install.py
drwxr-xr-x.  3 root root   62 Jan 24 22:21 _markerlib
drwxr-xr-x. 11 root root 4096 Jan 24 22:21 pip
drwxr-xr-x.  2 root root  154 Jan 24 22:21 pip-7.1.2.dist-info
drwxr-xr-x.  4 root root   59 Jan 24 22:21 pkg_resources
drwxr-xr-x.  2 root root   41 Jan 24 22:21 __pycache__
drwxr-xr-x.  4 root root  150 Jan 24 22:22 pytz
drwxr-xr-x.  2 root root  149 Jan 24 22:22 pytz-2018.9.dist-info
drwxr-xr-x.  4 root root 4096 Jan 24 22:21 setuptools
drwxr-xr-x.  2 root root  182 Jan 24 22:21 setuptools-18.2.dist-info
```
明明pip是把django安装在虚拟环境下面的，为什么Python不能正常导入呢？于是执行以下代码查看环境变量：
```
(test_venv) [root@localhost test_django]# python
Python 3.5.0 (default, Dec 28 2018, 00:17:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-36)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> print(sys.path)
['', '/usr/local/lib/python35.zip', '/usr/local/lib/python3.5', '/usr/local/lib/python3.5/plat-linux', '/usr/local/lib/python3.5/lib-dynload', '/root/.local/lib/python3.5/site-packages', '/usr/local/lib/python3.5/site-packages']
```
全部是系统下面Python的路径，和venv没有一点点的关系。  
在虚拟环境下面打印PATH：
```
(test_venv) [root@localhost test_django]# echo $PATH
/home/leich/Desktop/test_django/test_venv/bin:.:/usr/local/zk/bin:/usr/local/java/jdk1.8.0_191/bin:.:/usr/local/zk/bin:/usr/local/java/jdk1.8.0_191/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root/.local/bin
```
venv在环境变量的最前面，系统不应该是首先找环境变量第一个位置下面的Python吗？怎么会跳过虚拟环境，去打开了系统中的Python呢？应该直接打开虚拟环境下面的Python才对
## 问题分析
问题的根源就在你的alias上面。
zsh 的alias的优先级是非常高的，它会首先替换为等号后面的内容，然后再执行。那么即使在虚拟环境下，在终端输入python并回车以后，实际执行的代码是：  
```
/usr/local/bin/python3.5
```  
我们使用了绝对路径打开了系统中的Python3。  
而由于没有对pip设定alias, 因此在使用pip 安装django的时候，它调用的是虚拟环境下面的pip,所以django会正确安装在虚拟环境下面。  
## 解决问题
解决办法有两个:
1. 在/etc/profile.d/python.sh中删除下面的代码：  
```  
alias python='/usr/local/bin/python3.5'
```  
2. 将/etc/profile.d/python.sh中的：  
```
alias python='/usr/local/bin/python3.5'
``` 
修改为 
```
alias python=python3
```
1,2两种方法修改完成后执行以下命令，并重启终端：
```
source /etc/profile.d/python.sh
```
采用第2种修改方式，再次在命令行导入django包：
```
(test_venv) [root@localhost test_django]# python
Python 3.5.0 (default, Dec 28 2018, 00:17:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-36)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import django
>>> 
```
导入成功。再次执行命令创建test\_django数据库：
```
(test_venv) [root@localhost test_django]# python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying sessions.0001_initial... OK
```
创建成功。
  
