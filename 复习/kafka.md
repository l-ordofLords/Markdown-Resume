安装kafka

1.从官网下载安装包   curl -O 下载地址

2.tar -xvf 包

3.config/server.propertis 配置

listeners=PLAINTEXT://hostname:9092

advertised.listeners=PLAINTEXT://hostname:9092

kafka的hostname 去/etc/hosts里修改了修改成内网地址

然后把java调用所在的电脑或者服务器的hosts也改掉  改成hostname对应的外网地址（或者是能联通这个kafka服务器的ip地址）

4.假如出现消费者消费不了的情况，把topic    __consumer_offsets可以删除一下

idea有一个简单的kafka可视化插件   kafkalytic

