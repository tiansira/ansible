ansible facts组件是用来收集被管理节点信息的,使用ansible setup模块可以获取这些信息.

  ```
  aansible localhost -m setup

  ...
  ```

使用filter可以筛选指定的facts信息。例如：

```
ansible 192.168.100.64 -m setup -a "filter=changed"
192.168.100.64 | SUCCESS => {
    "ansible_facts": {}, 
    "changed": false
}

ansible localhost -m setup -a "filter=*ipv4"
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_default_ipv4": {
            "address": "192.168.100.62", 
            "alias": "eth0", 
            "broadcast": "192.168.100.255", 
            "gateway": "192.168.100.2", 
            "interface": "eth0", 
            "macaddress": "00:0c:29:d9:0b:71", 
            "mtu": 1500, 
            "netmask": "255.255.255.0", 
            "network": "192.168.100.0", 
            "type": "ether"
        }
    }, 
    "changed": false
}
```
facts收集的信息是json格式的，其内任一项都可以当作变量被直接引用(如在playbook、jinja2模板中)引用。见下文。

### 变量引用json数据的方式
在ansible中，任何一个模块都会返回json格式的数据，即使是错误信息都是json格式的。

在ansible中，json格式的数据，其内每一项都可以通过变量来引用它。当然，引用的前提是先将其注册为变量。

例如，下面的playbook是将shell模块中echo命令的结果注册为变量，并使用debug模块输出。

```
---
    - hosts: 192.168.100.65
      tasks:
        - shell: echo hello world
          register: say_hi
        - debug: var=say_hi
```
debug输出的结果如下：

```
TASK [debug] *********************************************
ok: [192.168.100.65] => {
    "say_hi": {
        "changed": true, 
        "cmd": "echo hello world", 
        "delta": "0:00:00.002086", 
        "end": "2017-09-20 21:03:40.484507", 
        "rc": 0, 
        "start": "2017-09-20 21:03:40.482421", 
        "stderr": "", 
        "stderr_lines": [], 
        "stdout": "hello world", 
        "stdout_lines": [
            "hello world"
        ]
    }
}
```

### 引用json字典数据的方式
如果想要输出json数据的某一字典项，则应该使用"key.dict"或"key['dict']"的方式引用。例如最常见的stdout项"hello world"是想要输出的项，以下两种方式都能引用该字典变量。
```
---
    - hosts: 192.168.100.65
      tasks:
        - shell: echo hello world
          register: say_hi
        - debug: var=say_hi.stdout
        - debug: var=sya_hi['stdout']
```
ansible-playbook的部分输出结果如下：

```
TASK [debug] ************************************************
ok: [192.168.100.65] => {
    "say_hi.stdout": "hello world"
}

TASK [debug] ************************************************
ok: [192.168.100.65] => {
    "say_hi['stdout']": "hello world"
}
```
"key.dict"或"key['dict']"的方式都能引用，但在dict字符串本身就包含"."的时候，应该使用中括号的方式引用。例如：
`anykey['192.168.100.65']`

### 引用json数组数据的方式
"ipv6": [
   {
       "address": "fe80::20c:29ff:fe26:1498", 
       "prefix": "64", 
       "scope": "link"
   }
]

其中key=ipv6，其内有且仅有是一个列表项，但该列表内包含了数个字典项。要引用列表内的字典，例如上面的address项。应该如下引用：

```
ipv6[0].address
```

### 引用facts数据

```
shell> ansible localhost -m setup -a "filter=*eth*"    
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_eth0": {
            "active": true, 
            "device": "eth0", 
            "features": {
                "busy_poll": "off [fixed]", 
                "fcoe_mtu": "off [fixed]", 
                "generic_receive_offload": "on", 
                .........................
            }, 
            "ipv4": {
                "address": "192.168.100.62", 
                "broadcast": "192.168.100.255", 
                "netmask": "255.255.255.0", 
                "network": "192.168.100.0"
            }, 
            "macaddress": "00:0c:29:d9:0b:71", 
            "module": "e1000", 
             ............................
        }
    }, 
    "changed": false
}
```

显然，facts数据的顶级key为ansible_facts，在引用时应该将其包含在变量表达式中。但自动收集的facts比较特殊，它以ansible_facts作为key，ansible每次收集后会自动将其注册为变量，所以facts中的数据都可以直接通过变量引用，甚至连顶级key ansible_facts都要省略。

例如引用上面的ipv4的地址address项。
```
ansible_eth0.ipv4.address
```
但其他任意时候，都应该带上所有的key。

### 设置本地facts

在ansible收集facts时，还会自动收集/etc/ansible/facts.d/*.fact文件内的数据到facts中，且以ansible_local做为key。目前fact支持两种类型的文件：ini和json。当然，如果fact文件的json或ini格式写错了导致无法解析，那么肯定也无法收集。
```
shell> ansible localhost -m setup -a "filter=ansible_local"
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_local": {
            "my": {
                "family": {
                    "father": {
                        "age": "39", 
                        "name": "Zhangsan"
                    }, 
                    "mother": {
                        "age": "35", 
                        "name": "Lisi"
                    }
                }
            }
        }
    }, 
    "changed": false
}
```
如果想要引用本地文件中的某个key，除了带上ansible_local外，还必须得带上fact文件的文件名。例如，引用father的name。
```
ansible_local.my.family.father.name
```

或者是ini格式的.

```
$ cat /etc/ansible/facts.d/test.fact
[dtian_car]
name = dtian
```

### 输出和引用变量

上文已经展示了一种变量的引用方式：使用debug的var参数。debug的另一个参数msg也能输出变量，且msg可以输出自定义信息，而var参数只能输出变量。

另外，msg和var引用参数的方式有所不同。例如：
```
---
    - hosts: 192.168.100.65
      tasks:
        - debug: 'msg="ipv4 address: {{ansible_eth0.ipv4.address}}"'
        - debug: var=ansible_eth0.ipv4.address
```
msg引用变量需要加上双大括号包围，既然加了大括号，为了防止被解析为内联字典，还得加引号包围。这里使用了两段引号，因为其内还包括了一个": "，加引号可以防止它被解析为"key: "的格式。而var参数引用变量则直接指定变量名。

这就像bash中引用变量的方式是一样的，有些时候需要加上$，有些时候不能加$。也就是说，当引用的是变量的值，就需要加双大括号，就像加$一样，而引用变量本身，则不能加双大括号。其实双大括号是jinja2中的分隔符。

执行的部分结果如下：
```
TASK [debug] *****************************************************
ok: [192.168.100.65] => {
    "msg": "ipv4 address: 192.168.100.65"
}

TASK [debug] *****************************************************
ok: [192.168.100.65] => {
    "ansible_eth0.ipv4.address": "192.168.100.65"
}
```
几乎所有地方都可以引用变量，例如循环、when语句、信息输出语句、template文件等等。只不过有些地方不能使用双大括号，有些地方需要使用。

### 注册和定义变量的各种方式


参考文档:
- https://www.cnblogs.com/f-ck-need-u/p/7571974.html
- 变量的调用顺序:https://www.cnblogs.com/mauricewei/p/10054300.html
