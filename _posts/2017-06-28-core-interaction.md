---
published: true
layout: post
title: Взаимодействие с прошивкой
---

<p class="message">
Есть несколько способов взаимодействия с ядром NDMS, сводящихся к эмуляции интерактивного CLI-сеанса. Вариант telnet-сессии со скриптами expect/empty отложу как совсем костыльный и опишу два других, имеющих свои достоинства и недостатки. 
</p>

### RESTful control interface

Достоинства:

* не требует ничего, кроме клиента, способного на http-запросы,
* машиночитаемый (спорно:).

Недостатки:

* требует digest-авторизацию, т.е. в скриптах надо указывать актуальные учётные данные пользователя с тегом `http`.

Обратите внимание на то, что апплет `wget` из Busybox digest-авторизацию не поддерживает, поэтому надо устанавливать полноценный `wget` или `curl`.

Ответы ядро присылает в виде JSON:
```
# curl -sX GET --digest http://admin:Passw0rd@127.0.0.1/rci/show/cloud
{
  "agent": {
    "service": {
      "state": "ACTIVE",
      "token_alias": "c0c004b7bcf6ed7bd12382a0381fc0c00",
      "since": 0,
      "loop_delay": 0,
      "loop_interval": 17,
      "loop_limit": 9,
      "loop_sleep": 30,
      "tcp_connect_timeout": 15,
      "serial_in": 0,
      "serial_out": 890,
      "direct_access": false,
      "target_host": "188.227.18.199",
      "target_port": "9",
      "target_list": "46.105.148.92, 91.218.112.172, 88.198.177.103, 188.227.18.199",
      "transport": "udp"
    }
  }
}
```
Следственно, для его разбора можно воспользоваться утилитой `jq`, имеющейся и в Entware и в Debian:
```
# curl -sX GET --digest http://admin:Passw0rd@127.0.0.1/rci/show/cloud | jq '.agent.service.target_host'
"188.227.18.199"
```

### Утилита ndmq

Достоинства:

* авторизации не требует,

Недостатки:

* требует предварительной сборки,
* в chroot-среде работает не через UNIX-socket, а через deprecated TCP-запросы.

Утилита зависит от `libndmq`, которую в случае с Debian сначала надо [собрать и установить](/2017/06/23/pam_ndm-auth/#%D1%81%D0%B1%D0%BE%D1%80%D0%BA%D0%B0-pam_ndm) и лишь затем собрать саму утилиту:

```
# cd ~
# git clone https://github.com/ndmsystems/ndmq.git
# cd ndmq
# make 
# strip ndmq
# cp ndmq /usr/local/bin
# # chown root:root /usr/local/bin/ndmq
```
Ключи вызова можно посмотреть здесь, ответ утилита возвращает в XML:
```
# ndmq -p 'show interface' -x 
<response>
    <interface name="GigabitEthernet0">
        <id>GigabitEthernet0</id>
        <index>0</index>
        <type>GigabitEthernet</type>
        <description/>
        <interface-name>GigabitEthernet0</interface-name>
        <link>down</link>
        <connected>no</connected>
        <state>up</state>
        <mtu>1500</mtu>
        <tx-queue>1000</tx-queue>
        <port name="1">
            <id>GigabitEthernet0/0</id>
            <index>0</index>
            <interface-name>1</interface-name>
            <type>Port</type>
            <link>down</link>
            <last-change>16151.087824</last-change>
            <last-overflow>0</last-overflow>
            <public>no</public>
        </port>
        <port name="2">
            <id>GigabitEthernet0/1</id>
            <index>1</index>
            <interface-name>2</interface-name>
            <type>Port</type>
            <link>down</link>
            <last-change>16151.086143</last-change>
            <last-overflow>0</last-overflow>
            <public>no</public>
...
```
Анализировать это в bash'e лучше не браться, в Debian и Entware для этого есть подходящая утилита `xmlstarlet`. В команде ниже отфильтрованы интерфейсами со свойствами `connected=yes`, `state=up`, `link=up` в виде таблички:
```
ndmq -p 'show interface' -x | xmlstarlet sel -t -m '//interface[link="up"][state="up"][connected="yes"]' -v '@name' -o ': ' -v 'address' -o '/' -v 'mask' -n
WifiMaster0/WifiStation0: 192.168.6.4/255.255.255.0
Home: 192.168.11.1/255.255.255.0
Guest: 10.1.30.1/255.255.255.0
```


