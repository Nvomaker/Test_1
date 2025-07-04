## 一、环境搭建

### 1、实验环境准备

准备三台主机，一台安装prometheus，一台安装grafana，一台为被监控的服务器

所有的服务器需要静态ip并且能够上外网，服务器配置主机名并绑定

```bash
#配置主机名
hostnamectl set-hostname --static k8s-master
#三台主机各自绑定ip和主机名
vim /etc/host
192.168.142.128 k8s-master
192.168.142.129 k8s-node1
192.168.142.130 k8s-node2
#配置服务器的时间
systemctl restart ntpd
systemctl enable ntpd
#服务器关闭防火墙与selinux
systemctl stop firewalld
systemctl disable firewalld
iptables -F
vim /etc/selinux/config
#把SELINUX=后面的内容改为disabled
```

### 2、安装prometheus

先对软件进行解压然后重命名一下看着舒服些，然后使用默认配置文件启动，并加上&后台符号

```bash
 tar -zxvf prometheus-3.4.2.linux-amd64.tar.gz -C /opt/module/
 mv /opt/module/prometheus-3.4.2.linux-amd64 /opt/module/prometheus-3.4.2
 #启动yml文件并且将日志放到nohup.out文件中方便查看
 nohup /opt/module/prometheus-3.4.2/prometheus --
config.file="/opt/module/prometheus-3.4.2/prometheus.yml" &
```

可以使用命令查看prometheus的端口情况

```bash
netstat -tunlp | grep prometheus
```

 ![image-20250704155704241](https://github.com/user-attachments/assets/fa89d07c-51ab-439c-ab9c-089f6937762b)


在浏览器输入http://192.168.142.128:9090即可进入prometheus的界面

![image-20250704155901704](https://github.com/user-attachments/assets/7cd00456-e531-4b17-8b29-bca8261f00da)

可以看到目前的监控目标

![image-20250704160021312](https://github.com/user-attachments/assets/d470d5c6-3689-482c-83d6-189e1704cbb1)



## 二、监控linux主机

### 1、安装node_exporter并启动服务

```bash
tar -zxvf node_exporter-1.9.1.linux-amd64.tar.gz -C /opt/module/
mv node_exporter-1.9.1.linux-amd64/ node_exporter-1.9.1
nohup /opt/module/node_exporter-1.9.1/node_exporter &
```

### 2、验证相应的9100端口

先到master主机上使用kill关闭掉prometheus进程

然后修改文件prometheus的yml文件

```bash
vim /opt/module/prometheus-3.4.2/prometheus.yml
#添加以下内容对node1服务器进行监控
- job_name: "k8s-node1"
    static_configs:
      - targets: ["192.168.142.129:9100"]
#修改完成后再重新启动进程
nohup /opt/module/prometheus-3.4.2/prometheus --config.file="/opt/module/prometheus-3.4.2/prometheus.yml" &
```

![image-20250704163129688](https://github.com/user-attachments/assets/eda8f578-6562-4614-995e-b46329283665)

可以看到node1服务器在监控的目标里



## 三、监控mysql服务器

### 1、在node2上启动mariadb

```bash
systemctl start mariadb
systemctl enable mariadb
```

如果启动报错可能是端口被占用，使用netstat -tulnp | grep 3306查看

![image-20250704165525831](https://github.com/user-attachments/assets/3d9e43a3-cd7c-4120-a5ca-47581b2215b1)

### 2、授权

mariadb授权方式：

```sql
mysql -uroot -p
MariaDB [(none)]> grant all ON *.* to 'mysql_master'@'localhost' identified by '123';
MariaDB [(none)]> flush privileges;
MariaDB [(none)]> exit
```

MYSQL8s授权方式：

```sql
mysql> create user 'mysql_master'@'%' identified with mysql_native_password by '123';
mysql> grant all on *.* to 'mysql_master'@'%';
mysql> flush privileges;
mysql> quit
```



### 3、安装mysqld_exporter组件

```bash
tar -zxvf mysqld_exporter-0.17.2.linux-amd64.tar.gz -C /opt/module/
mv mysqld_exporter-0.17.2.linux-amd64/ mysqld_exporter-0.17.2
vim /opt/module/mysqld_exporter-0.17.2/.my.cnf 
#添加客户端的内容
[client]
user=mysql_master
password=123
#启动mysqld_exporter
nohup /opt/module/mysqld_exporter-0.17.2/mysqld_exporter --config.my-cnf='/opt/module/mysqld_exporter-0.17.2/.my.cnf' &
```

可以在192.168.142.129:9104/metrics查看到相应的一些配置信息

![image-20250704172223497](https://github.com/user-attachments/assets/a3c8c06c-c6f3-4ccc-a107-039db04c7476)

### 4、让master主机能够监控node2

先到master主机上使用kill关闭掉prometheus进程

然后修改文件prometheus的yml文件

```bash
vim /opt/module/prometheus-3.4.2/prometheus.yml
#添加以下内容对node2服务器进行监控
- job_name: "k8s-node1_mariadb"
    static_configs:
      - targets: ["192.168.142.129:9104"]
#修改完成后再重新启动进程
nohup /opt/module/prometheus-3.4.2/prometheus --config.file="/opt/module/prometheus-3.4.2/prometheus.yml" &
```

可以在prometheus中看到监控目标已加入

![image-20250704174043755](https://github.com/user-attachments/assets/e8699dce-97a2-455d-a78d-ef1b960e4dcf)



## 四、grafana数据可视化

### 1、下载granfana的rpm包并安装

```bash
yum install -y grafana-12.0.2-1.x86_64.rpm 
#启动服务
systemctl start grafana-server
systemctl enable grafana-server
#验证端口
netstat -nltup | grep :3000
```

在浏览器输入node2的地址和Grafana使用的端口号后就能进入Grafana的登录页面了，输入默认账号密码admin即可进入

![image-20250704203630560](https://github.com/user-attachments/assets/5a2ca28d-64bd-4289-b87c-9716e97b0622)

### 2、设置prometheus为Grafana的数据源

![image-20250704205634069](https://github.com/user-attachments/assets/5c2cdce9-3006-450b-970a-3acf868c7000)

![image-20250704214834223](https://github.com/user-attachments/assets/2728f8fc-ad98-4d41-a560-88953711cc78)

![image-20250704215055538](https://github.com/user-attachments/assets/da563394-924f-4419-9063-8d4ddac334f1)

### 3、自定义模板

![image-20250704231850790](https://github.com/user-attachments/assets/c4aa48f1-a7b4-47b9-a4d8-f45f37aa74e5)

### 4、引入Grafana提供的官方模板

在仪表盘位置找到import进行引入

![image-20250704232035266](https://github.com/user-attachments/assets/728f1840-80f0-4997-904f-8947a6f3a06f)

找到合适的模板复制模板ID进行引入

![image-20250704232206329](https://github.com/user-attachments/assets/612effef-5b49-49da-a1be-84a0964b6103)

填入ID后载入

![image-20250704232510757](https://github.com/user-attachments/assets/41a245db-57e4-41bb-a735-e24c88c4c01a)

引入之后得到相应的仪表盘

![image-20250704232607996](https://github.com/user-attachments/assets/20b9adc7-e2e7-4f34-be4f-f2348c0f0e03)

使用的mysql监控模板也能显示各性能参数情况

![image-20250704234046032](https://github.com/user-attachments/assets/57da3c75-4d3a-436c-bde3-87232e1197f9)

