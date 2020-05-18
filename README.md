## ansible playbook 搭建phpmyadmin web集群

一些参考模块

https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html

https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html

### 1环境搭建

```
[root@m01 ~]# ssh-keygen -t rsa -C ansible-use
[root@m01 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@10.0.0.7
[root@m01 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@10.0.0.8
...

[root@m01 ~]# yum install -y ansible
[root@m01 ~]# mkdir -p /perject-php/{redis,webservers,lbs,php}
[root@m01 ~]# cd /perject-php

[root@m01 /perject-php]# vim ansible.cfg
[defaults]
inventory  = /perject-php/hosts

[root@m01 /perject-php]# vim hosts
[db]
172.16.1.51
[redis]
172.16.1.71
[Webservers]
172.16.1.7
172.16.1.8
[lbs]
172.16.1.5

[root@m01 /perject-php]# tree /perject-php/
/perject-php/
├── ansible.cfg
├── db.yaml
├── hosts
├── lbs
│   └── proxy.php.conf.j2
├── lbs.yaml
├── php
│   ├── php.ini.j2
│   └── www.conf.j2
├── phpMyAdmin-5.0.2-all-languages.zip
├── redis
│   └── redis.conf.j2
├── redis.yaml
├── webservers
│   ├── config.inc.php.j2
│   ├── nginx.conf.j2
│   └── php.com.conf.j2
└── web.yaml
```



### 2数据库剧本编写

```
- hosts: db
  tasks:
  - name: yum mariadb and mariadb-server
    yum:
      name: "{{ packages }}"
      state: present
    vars:
      packages:
        - mariadb
        - mariadb-server
        - MySQL-python
  - name: enabled and start mariadb
    systemd:
      name: mariadb
      state: restarted
      enabled: yes
  - name: create database php
    mysql_db:
      name: php
      state: present
  - name: create mysql user for phpmyadmin
    mysql_user:
      name: phpmyadmin
      password: 123456
      priv: '*.*:ALL'
      host: '172.16.1.%'
      state: present
```



### 3redis缓存剧本编写

```
- hosts: redis
  tasks:
  - name: yum redis
    yum: 
      name: redis
      state: present
  - name: enable and restart redis
    systemd:
      name: redis
      state: restarted
      enabled: yes
  - name: cp redis file
    copy:
      src: ./redis/redis.conf.j2
      dest: /etc/redis.conf
      owner: redis
      group: root
      mode: 0640
      backup: yes
    notify: restart redis

handlers:
  - name: restart redis
    systemd:
      name: redis
      state: restarted
```



### 4web服务剧本编写

```
- hosts: Webservers
  tasks:
  - name: yum nginx php
    yum: 
      name: "{{ packages }}"
      state: present
    vars:
      packages:
        - nginx
        - php71w
        - php71w-cli
        - php71w-common
        - php71w-devel
        - php71w-embedded
        - php71w-gd
        - php71w-mcrypt
        - php71w-mbstring
        - php71w-pdo
        - php71w-xml
        - php71w-fpm
        - php71w-mysqlnd
        - php71w-opcache
        - php71w-pecl-memcached
        - php71w-pecl-redis
        - php71w-pecl-mongodb

  - name: groupadd www
    group:
      name: www
      gid: 666
  - name: useradd www
    user:
      name: www
      uid: 666
      group: www
      shell: /sbin/nologin
      create_home: no
  - name: mkdir /code
    file:
      path: /code
      state: directory
      owner: www
      group: www
      mode: 0755
      recurse: yes
  - name: copy/unarchive phpmyadmin web file
    unarchive:
      src: ./phpMyAdmin-5.0.2-all-languages.zip
      dest: /code/
      owner: www
      group: www

  - name: copy phpmyadmin_config.inc.php.j2 file
    copy:
      src: ./webservers/config.inc.php.j2
      dest: /code/phpMyAdmin-5.0.2-all-languages/config.inc.php
      owner: www
      group: www
      mode: 0644
      backup: yes
    notify: restart nginx		
  - name: copy /etc/php.ini config file
    copy:
      src: ./php/php.ini.j2
      dest: /etc/php.ini
      backup: yes
    notify: restart php-fpm		
  - name: copy /etc/php-fpm.d/www.conf config file
    copy:
      src: ./php/www.conf.j2
      dest: /etc/php-fpm.d/www.conf
      backup: yes
    notify: restart php-fpm
  - name: copy /etc/nginx/nginx.conf
    copy:
      src: ./webservers/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      backup: yes
    notify: restart nginx   
  - name: copy php.com.conf
    copy:
      src: ./webservers/php.com.conf.j2
      dest: /etc/nginx/conf.d/php.com.conf
      backup: yes
    notify: restart nginx

handlers:
  - name: restart php-fpm
    systemd: 
      name: php-fpm
      state: restarted
      enabled: yes		
  - name: restart nginx
    systemd: 
      name: nginx
      state: restarted
      enabled: yes
```



### 5 lbs负载均衡剧本编写

```
- hosts: lbs
  tasks:
  - name: yum nginx
    yum:
      name: nginx
      state: present
  - name: cp lb-conf file
    copy:
      src: ./lbs/proxy.php.conf.j2
      dest: /etc/nginx/conf.d/proxy.php.conf
      backup: yes
    notify: restart lb-nginx	  

handlers:
  - name: restart lb-nginx  
    systemd:
    name: nginx
    state: restarted
    enabled: yes
```

