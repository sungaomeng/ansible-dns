# ansible-dns (bind9主从版)
### 目录结构
```
[root@test-ansible002 ansible-dns]# tree
.
├── ansible.cfg
├── deploy.yml
├── hosts
├── README.md
└── roles
    ├── dns-master
    │   ├── files
    │   │   └── seetestdns.com.zone
    │   ├── handlers
    │   │   └── main.yaml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       ├── named.conf
    │       └── named.rfc1912.zones
    └── dns-slave
        ├── files
        ├── handlers
        │   └── main.yaml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── named.conf
            └── named.rfc1912.zones

11 directories, 13 files
```
### dns配置文件说明
```
    /etc/named.conf：    #主配置文件，用于定义dns的全局配置，include其他配置文件片段
        acl ACL_NAME {        //用来定义ip或网络地址列表,后面配置需要应用该ip列表时可以通过ACL_NAME来引用
            1.1.1.1;
            192.168.0.0/24;
            ...
        }；
        options {
	    listen-on port 53 { any; };     //指定dns服务的监听地址和端口，any表示在本机所有ip上监听53端口
	    directory 	"/var/named";      //指定区域解析数据库文件的根目录
            notify yes;                    //master dns有修改时立即通知slave
	    allow-query     { any; };       //设置允许哪些client做dns查询
	    forward         only;        //开启转发dns功能，可选项有first|only，first表示先让转发的dns服务器解析，如果没有得到结果，再由本地dns解析；only表示仅让转发目标dns去解析
	    forwarders      { 
        	114.114.114.114;      //转发的目标dns地址列表
                8.8.8.8;
            };
	    allow-recursion { ACL_NAME; };     //指定接收递归的客户端ip列表
	    dnssec-enable no;         //协议安全加密通信相关的，一般关闭
	    dnssec-validation no;     //协议安全加密通信相关的，一般关闭
        };
        logging {       //日志记录相关的
         channel "named_log" {
          file "/var/log/named/named.log" versions 10 size 20m;
          severity info;
          print-category yes;
          print-severity yes;
          print-time yes;
         };
         category default { named_log; };
        };
       include "/etc/named.rfc1912.zones";       //包含其他配置文件
       include "/etc/named.root.key";

    /etc/named.rfc1912.zones：      #被主配置文件包含，一般用于定义每个zone的类型、zone数据库文件的路径及相关的安全策略
        view VIEW_NAME {                         //设置视图，用于根据客户端源IP来选在不同的解析。如果使用了视图，那么所有的zone配置必须位于视图之内
	    match-clients {                          //指定匹配的客户端ip列表
		    internal;
	    };
	    zone "." IN {                               //定义根区域的配置
		    type hint;                           //根区域的类型为hint
		    file "named.ca";                //根区域的数据库文件
	    };
	    zone "gateray.org" IN {            //自定义区域
		    type master;                   //类型为master，表示为该区域的主dns
		    file "seetestdns.com.zone";      //指定区域数据库文件的路径
		    allow-update { none; };    //指定允许哪些ip可以动态的更新区域数据的资源记录，none表示不允许更新
		    allow-transfer { slaves; };    //允许向哪些从dns服务器返回区域数据更新
	    };
        
        };

    /var/named/seetestdns.com.zone：        #保存dns资源解析记录的文件数据库，一般以ZONE_NAME.zone方式命名
        $TTL 1D
        @		IN  SOA   @    admin (
                        2019021101           ;更新的序列号
                        2H                   ;主从同步间隔时长
                        10M                  ;同步失败的重试间隔
                        1W                   ;主dns挂掉时，从服务器能够提供服务的最大时长
                        6H )                 ;用于告诉客户端，将解析失败的结果缓存多久
                           IN  NS    ns1     ;NS记录
                           IN  NS    ns2            
        ns1             IN  A     192.168.1.68     ;权威dns的A记录
        ns2             IN  A     192.168.1.69
        www             IN  A     192.168.1.68     ;主机A记录
        m               IN  CNAME www              ;主机别名记录
        *               IN  A     192.168.1.69     ;泛域名A记录
```
### 使用方法
```
yum -y install ansible
git clone https://github.com/see-sgm/ansible-dns.git
cd ansible-dns
 #修改hosts文件
 #1.修改masters/slaves主机地址
 #2.修改ssh账户密码
 #3.修改转发dns地址/zones等dns参数
 #将roles/dns-master/files/seetestdns.com.zone 更改为自己的zones文件和内容
ansible all -m ping
ansible-playbook deploy.yml
```
### 执行过程如下
```
[root@test-ansible002 ansible-dns]# ansible-playbook deploy.yml

PLAY [masters] ******************************************************************************************

TASK [dns-master : Install bind9] ***********************************************************************
[DEPRECATION WARNING]: Invoking "yum" only once while using a loop via squash_actions is deprecated.
Instead of using a loop to supply multiple items and specifying `name: "{{ item }}"`, please use `name:
['bind', 'bind-libs', 'bind-utils', 'libselinux-python']` and remove the loop. This feature will be
removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in
ansible.cfg.
changed: [192.168.1.68] => (item=[u'bind', u'bind-libs', u'bind-utils', u'libselinux-python'])

TASK [dns-master : Create log dir] **********************************************************************
changed: [192.168.1.68]

TASK [dns-master : Config to bind9] *********************************************************************
changed: [192.168.1.68] => (item=named.conf)
changed: [192.168.1.68] => (item=named.rfc1912.zones)

TASK [dns-master : Add zone databases] ******************************************************************
changed: [192.168.1.68] => (item=seetestdns.com)

RUNNING HANDLER [dns-master : Restart named daemon] *****************************************************
changed: [192.168.1.68]

RUNNING HANDLER [dns-master : Reload named daemon] ******************************************************
changed: [192.168.1.68]

PLAY [slaves] *******************************************************************************************

TASK [dns-slave : Install bind9] ************************************************************************
[DEPRECATION WARNING]: Invoking "yum" only once while using a loop via squash_actions is deprecated.
Instead of using a loop to supply multiple items and specifying `name: "{{ item }}"`, please use `name:
['bind', 'bind-libs', 'bind-utils', 'libselinux-python']` and remove the loop. This feature will be
removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in
ansible.cfg.
changed: [192.168.1.69] => (item=[u'bind', u'bind-libs', u'bind-utils', u'libselinux-python'])

TASK [dns-slave : Create log dir] ***********************************************************************
changed: [192.168.1.69]

TASK [dns-slave : Config to bind9] **********************************************************************
changed: [192.168.1.69] => (item=named.conf)
changed: [192.168.1.69] => (item=named.rfc1912.zones)

RUNNING HANDLER [dns-slave : Restart named daemon] ******************************************************
changed: [192.168.1.69]

PLAY RECAP **********************************************************************************************
192.168.1.68               : ok=6    changed=6    unreachable=0    failed=0
192.168.1.69               : ok=4    changed=4    unreachable=0    failed=0
```
