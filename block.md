# block用法

1. 使用block对tasks进行分组. 
when条件将在运行block中的三个task之前被判断,三个任务都使用become root. ignore:errors: true确保Ansible继续执行playbook,即使一些任务失败.

```
tasks:
   - name: Install, configure, and start Apache
     block:
       - name: Install httpd and memcached
         ansible.builtin.yum:
           name:
           - httpd
           - memcached
           state: present

       - name: Apply the foo config template
         ansible.builtin.template:
           src: templates/src.j2
           dest: /etc/foo.conf

       - name: Start service bar and enable it
         ansible.builtin.service:
           name: bar
           state: started
           enabled: True
     when: ansible_facts['distribution'] == 'CentOS'
     become: true
     become_user: root
     ignore_errors: true
```

我们建议在所有的task中使用name，不管是在block内还是在其他地方，以便在运行playbook时更好地了解正在执行的task。

---

2. 使用block来处理错误.
你可以使用带有rescue和always部分的块来控制Ansible对任务错误的响应方式。
rescue指定了当一个块中的早期任务失败时要运行的任务。这种方法类似于许多编程语言中的异常处理。Ansible只在一个任务返回 "失败 "状态后运行rescue块。错误的的任务定义和无法到达的主机将不会触发rescue块。

```
tasks:
 - name: Handle the error
   block:
     - name: Print a message
       ansible.builtin.debug:
         msg: 'I execute normally'

     - name: Force a failure
       ansible.builtin.command: /bin/false

     - name: Never print this
       ansible.builtin.debug:
         msg: 'I never execute, due to the above task failing, :-('
   rescue:
     - name: Print when errors
       ansible.builtin.debug:
         msg: 'I caught an error, can do stuff here to fix it, :-)'
```

您还可以将 always 部分添加到块中。无论前一个块的任务状态如何，always 部分中的任务都会运行。
```
- name: Always do X
   block:
     - name: Print a message
       ansible.builtin.debug:
         msg: 'I execute normally'

     - name: Force a failure
       ansible.builtin.command: /bin/false

     - name: Never print this
       ansible.builtin.debug:
         msg: 'I never execute :-('
   always:
     - name: Always do this
       ansible.builtin.debug:
         msg: "This always executes, :-)"
```

这些元素结合在一起，提供了复杂的错误处理。

```
- name: Attempt and graceful roll back demo
  block:
    - name: Print a message
      ansible.builtin.debug:
        msg: 'I execute normally'

    - name: Force a failure
      ansible.builtin.command: /bin/false

    - name: Never print this
      ansible.builtin.debug:
        msg: 'I never execute, due to the above task failing, :-('
  rescue:
    - name: Print when errors
      ansible.builtin.debug:
        msg: 'I caught an error'

    - name: Force a failure in middle of recovery! >:-)
      ansible.builtin.command: /bin/false

    - name: Never print this
      ansible.builtin.debug:
        msg: 'I also never execute :-('
  always:
    - name: Always do this
      ansible.builtin.debug:
        msg: "This always executes"
```

block中的任务正常执行。如果block中的任何task返回失败，rescue部分将执行任务以恢复错误。无论block和rescue部分的结果如何，always部分都会运行。

如果block中发生错误并且rescue task运行成功,ansible将原任务的失败状态恢复为运行,并继续运行该play,就像原始任务成功了一样.
rescue的任务被认为是成功的，并且不会触发 max_fail_percentage 或 any_errors_fatal 配置。然而，Ansible仍然会在playbook统计中报告一个失败。

你可以在rescue任务中使用带有flush_handlers的块，以确保所有处理程序的运行，即使发生错误：

```
tasks:
   - name: Attempt and graceful roll back demo
     block:
       - name: Print a message
         ansible.builtin.debug:
           msg: 'I execute normally'
         changed_when: true
         notify: run me even after an error

       - name: Force a failure
         ansible.builtin.command: /bin/false
     rescue:
       - name: Make sure all handlers run
         meta: flush_handlers
 handlers:
    - name: Run me even after an error
      ansible.builtin.debug:
        msg: 'This handler runs even on error'
```
