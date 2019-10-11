
- name: symlink
  hosts: all
  tasks:
  - name: Check that the file link exists
    become: yes
    become_user: root
    stat:
      path: /home/centos/raj1
    register: stat_result
  - fail:
     msg: "Whoops! file ownership has changed"
    when: stat_result.stat.pw_name != 'root'

  - name: Create the file, if it doesnt exist already
    become: yes
    become_user: root
    file: path=/home/centos/raj1
           src=/home/centos/ansble/test1
           state=link
           force=yes
    when: stat_result.stat.exists == False

