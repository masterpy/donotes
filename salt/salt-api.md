##salt-api

###前言###

salt-api安装以及基础调用

***
####初始化salt-api

>初始化yum源

	rpm -Uvh http://mirrors.hust.edu.cn/epel//6/x86_64/epel-release-6-8.noarch.rpm

>安装

	yum install salt-master salt-api

>初始密钥

	[root@rs2 ~]# cd /etc/pki/tls/certs
	[root@rs2 certs]# make testcert
	umask 77 ; \
		/usr/bin/openssl genrsa -aes128 2048 > /etc/pki/tls/private/localhost.key
	Generating RSA private key, 2048 bit long modulus
	.....+++
	........+++
	e is 65537 (0x10001)
	Enter pass phrase:  [加密短语]
	Verifying - Enter pass phrase: [加密短语]
	umask 77 ; \
		/usr/bin/openssl req -utf8 -new -key /etc/pki/tls/private/localhost.key -x509 -days 365 -out /etc/pki/tls/certs/localhost.crt -set_serial 0
	Enter pass phrase for /etc/pki/tls/private/localhost.key: [加密短语]
	You are about to be asked to enter information that will be incorporated
	into your certificate request.
	What you are about to enter is what is called a Distinguished Name or a DN.
	There are quite a few fields but you can leave some blank
	For some fields there will be a default value,
	If you enter '.', the field will be left blank.
	-----
	Country Name (2 letter code) [XX]:ZH
	State or Province Name (full name) []:AH
	Locality Name (eg, city) [Default City]:HF
	Organization Name (eg, company) [Default Company Ltd]:tbcs
	Organizational Unit Name (eg, section) []:tbc
	Common Name (eg, your name or your server's hostname) []:cap
	Email Address []: getcat@163.com


	#将私钥转化为无密码访问形式
	[root@rs2 certs]# cd /etc/pki/tls/private/
	[root@rs2 private]#  openssl rsa -in localhost.key -out localhost_nopass.key


>添加salt-api模块配置
	
salt-master默认配置文件/etc/salt/master默认配置如下：

	# Per default, the master will automatically include all config files
	# from master.d/*.conf (master.d is a directory in the same directory
	# as the main master config file).
	#default_include: master.d/*.conf

便于管理与修改，创建此目录添加对应自定义配置：
	
	mkdir /etc/salt/master.d
	
	[root@rs2 master.d]# cat api.conf 
	rest_cherrypy:
	  port: 8000
	  ssl_crt: /etc/pki/tls/certs/localhost.crt
	  ssl_key: /etc/pki/tls/private/localhost_nopass.key

	[root@rs2 master.d]# cat eauth.conf 
	external_auth:
	  pam:
	    saltapi:
	      - .*
	    user1:
		  - .*

>启动或重启

	/etc/init.d/salt-master start
	/etc/init.d/salt-api start

>端口查看

	lsof -i:8000


###调用测试

>访问测试
	
	[root@rs1 ~]# curl -ik https://192.168.100.83:8000/login -H "Accept: application/x-yaml"  

	HTTP/1.1 200 OK
	Content-Length: 35
	Access-Control-Expose-Headers: GET, POST
	Vary: Accept-Encoding
	Server: CherryPy/3.2.2
	Allow: GET, HEAD, POST
	Access-Control-Allow-Credentials: true
	Date: Mon, 07 Sep 2015 02:37:39 GMT
	Access-Control-Allow-Origin: *
	Content-Type: application/x-yaml
	Www-Authenticate: Session
	Set-Cookie: session_id=e77f8d179509bd8f680b64047d45ae77fb2f7b71; expires=Mon, 07 Sep 2015 12:37:39 GMT; Path=/
	
	return: Please log in
	status: null

>登陆获取token

	[root@rs2 ~]# curl -k https://192.168.100.83:8000/login -H "Accept: application/x-yaml"  -d username='saltapi' -d password='saltapi.123' -d eauth='pam'
	
	return:
	- eauth: pam
	  expire: 1441636751.138237
	  perms:
	  - .*
	  start: 1441593551.138236
	  token: 0052d8e47083305364b1b08108746e03f07ac434
	  user: saltapi

>使用token进行通信

	[root@rs2]# curl -k https://192.168.100.83:8000/ -H "Accept: application/x-yaml" -H "X-Auth-Token: 0052d8e47083305364b1b08108746e03f07ac434" -d client='local' -d tgt='*' -d fun='test.echo' -d arg='hello world'

	return:
	- rs1: hello world


####封装salt-api的操作

>类封装

	#!/usr/bin/env python
	#coding=utf-8
	 
	import urllib2, urllib, json, re
	 
	class saltAPI:
	    def __init__(self):
	        self.__url = 'https://192.168.100.83:8000'       #salt-api监控的地址和端口如:'https://192.168.100.83:8000'
	        self.__user =  'saltapi'             #salt-api用户名
	        self.__password = 'password'          #salt-api用户密码
	        self.__token_id = self.salt_login()
	 
	    def salt_login(self):
	        params = {'eauth': 'pam', 'username': self.__user, 'password': self.__password}
	        encode = urllib.urlencode(params)
	        obj = urllib.unquote(encode)
	        headers = {'X-Auth-Token':''}
	        url = self.__url + '/login'
	        req = urllib2.Request(url, obj, headers)
	        opener = urllib2.urlopen(req)
	        content = json.loads(opener.read())
	        try:
	            token = content['return'][0]['token']
	            return token
	        except KeyError:
	            raise KeyError
	 
	    def postRequest(self, obj, prefix='/'):
	        url = self.__url + prefix
	        headers = {'X-Auth-Token'   : self.__token_id}
	        req = urllib2.Request(url, obj, headers)
	        opener = urllib2.urlopen(req)
	        content = json.loads(opener.read())
	        return content['return']
	 
	    def saltCmd(self, params):
	        obj = urllib.urlencode(params)
	        obj, number = re.subn("arg\d", 'arg', obj)
	        res = self.postRequest(obj)
	        return res
	 
	def main():
	    #以下是用来测试saltAPI类的部分
	    sapi = saltAPI()
 		params = {'client':'local', 'fun':'test.echo', 'tgt':'*', 'arg1':'hello'}
	    
		#params = {'client':'local', 'fun':'test.ping', 'tgt':'*'}
	    #params = {'client':'local', 'fun':'test.ping', 'tgt':'某台服务器的key'}   
	    #params = {'client':'local', 'fun':'test.ping', 'tgt':'某组服务器的组名', 'expr_form':'nodegroup'}
	    test = sapi.saltCmd(params)
	    print test
	 
	if __name__ == '__main__':
	    main()
	

>运行测试

	[root@rs2 ~]# python test.py 
	[{u'rs1': u'hello'}]

	