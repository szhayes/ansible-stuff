# Problem Solving a New Automation

You are in the middle of a cloud migration and you have to enable an existing account on 400+ servers as quickly as possible.   You don't want to create the account on a server if it is not already there but just enable an existing, disabled account for a period of six months.   You know Ansible has a module for this.  So a quick trip to your favourite editor, a couple of Google searches later, you have an approach.

Let's write some code! To start our development we are going to write our code and test against our localhost.

>`code ./addauser.yml`

```
---
- name: add a user
  hosts: localhost
  gather_facts: false
  become: yes

  tasks:
  - name: Add a user with no expiry
    user:
      name: pentest
      expires: -1
```
Ok, ok, lets test our code.
>`ansible-playbook addauser.yml`

```
$ sudo chage -l pentest
[sudo] password for stuart:            
chage: user 'pentest' does not exist in /etc/passwd

$ ansible-playbook 1-addauser.yml --ask-become
BECOME password: 
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [add a user] *****************************************************************************************

TASK [Add a user with no expiry] *****************************************************************************************
changed: [localhost]

PLAY RECAP *****************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

$ sudo chage -l pentest
Last password change                                    : Apr 26, 2022
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

So what happened here:

1) user pentest did not exist, and we proved that with the `chage -l pentest`
2) executed the playbook and a new user was created with no account expiry 
3) to keep a privileged password safe, prompt for the privileged password with **--ask-become**
4) we saved some time by not gathering facts

But we failed, against our requirements, we created an account if it did not exist.  So how can we prevent a user from being created if it already exists?  We are going to need to somehow check if the user is already in our database of existing users.

Quick search of the interwebs suggest that we can do a getent.  For more information check out [GetEnt on Geeks for Geeks](https://www.geeksforgeeks.org/getent-command-in-linux-with-examples/).

>`getent passwd not-user` 

at the command line would return the following:
```
$ getent passwd not-user
$ echo $?
2
```
The command returns no output but appears to work successfully.  Just a quick look at the return code, we find that RC of 2 is returned. From the **getent(1)** man page:

>**Exit Status**

>| 2 | One or more supplied key could not be found in the database. |

So can we write a task that we can add to our playbook that checks for the existance of the user we are looking for?  Yes Ansible has a module that supports **getent(1)** you can find the documentation at:
[ansible.builtin.getent](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/getent_module.html)

```
  - name: GETENT pentest info
    getent:
      database: passwd
      key: pentest
    ignore_errors: yes
```

There is a little bit of a 'fail' here on my part but we will come back to that later.  We **ignore_errors:**  because by default the getent module will fail if the key (pentest in our case) does not exist in the database (passwd) we are seaching.  Teaser:  read the entire manual page and understand the options for a future improvement!

The getent module returns a FACT that is stored in ansible_facts.genent_*DBNAME[key]*.  In our example above this is ansible_facts.getent_passwd[pentest].  Combining our fist code we can now determine if the user we want to update, exists. If it does we can then update the account expiry time with a value of -1 to set to never expire.

```
---
- name: add a user
  hosts: localhost
  gather_facts: false
  become: yes

  tasks:
  - name: GETENT pentest info
    getent:
      database: passwd
      key: pentest
    ignore_errors: yes 

  - name: Add user pentest with no expiry
    user:
      name: pentest
      expires: -1
    when: ansible_facts.getent_passwd[pentest] is defined
```

So are we there yet?  Did we meet our goals:  enable an existing, disabled account for a period of six months.  Check, err no.  Ok close but not there yet, let's keep going.

There are several ways to get or set the expires date.   And I fooled around with a couple,  you could register a variable or set_fact but these would have to be executed for each host.  So let's set a var because six months from now set for all hosts will be good AND for extra fanciness lets set another var with the user name and make our code reusuable for any user.  We can always pass in variables from the command to change the user and the expiry time.

```
---
- name: update users
  hosts: localhost
  gather_facts: false
  become: yes

  vars:
    s_user: 'pentest'
    s_expires: "{{ lookup('pipe','date +%s -d now+6months') }}"

  tasks:
  - name: GETENT {{s_user}} info
    getent:
      database: passwd
      key: "{{ s_user }}"
    ignore_errors: yes

  - name: Change expiry date for user to today + 6 months
    user:
      name: "{{ s_user }}"
      # expires: -1
      expires: "{{ s_expires }}"
    when: ansible_facts.getent_passwd["{{s_user}}"] is defined
```
There are lots of best practices around the selection of variable names.   A couple of good ones, don't make them cryptic, do make them stick out and not like module names.

What happens now?

```
$ ansible-playbook adduser.yml --ask-become
BECOME password: 
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [update users] **************************************************************************************************

TASK [GETENT pentest info] **************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "One or more supplied key could not be found in the database."}
...ignoring

TASK [Change expiry date for user to today + 6 months] **************************************************************************************************
[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: ansible_facts.getent_passwd["{{s_user}}"] is defined
skipping: [localhost]

PLAY RECAP **************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=1
```

Okay our getent task failed because we could not find the relavent key in the passwd database, but we ignore_errors on that so we keep going.  Again if we have multiple hosts we don't want to fail on a single error.   Then our user task is skipped because the WHEN conditional is not met, because we did not create a ansible_facts.getend_passwd[pentest] variable eg. it is not defined.

Quick confirmation:
```
$ sudo chage -l pentest
chage: user 'pentest' does not exist in /etc/passwd
```

Okay we can add the user either by command line or by creating a quick and dirty playbook.  Your choice. Expire the account for some time before now.  

```
$ sudo usermod -e 2022-01-01 pentest
$ sudo chage -l pentest
Last password change                                    : Apr 27, 2022
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : Jan 01, 2022
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```
With the expired user present lets run our playbook again.

```
$ ansible-playbook adduser.yml --ask-become
BECOME password: 
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [update users] **************************************************************************************************

TASK [GETENT pentest info] **************************************************************************************************
ok: [localhost]

TASK [Change expiry date for user to today + 6 months] **************************************************************************************************
[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: ansible_facts.getent_passwd["{{s_user}}"] is defined
changed: [localhost]

PLAY RECAP **************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

$ sudo chage -l pentest
Last password change                                    : Apr 27, 2022
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : Oct 27, 2022
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

Success!  We have met our goals:  enable an existing, disabled account for a period of six months.  CHECK and CHECK.  We created a reusable playbook because we can override the variable s_user and s_expires to choose different users and a different expiry date.

One final update update the hosts: line to operate against all hosts
> hosts: all

Wait!, one more thing, remember the small 'fail' above?  And that was a hint. I asked a colleague to check my work and suggest some improvements if they had any.  They cautioned about the use of ignore_errors, if there is another option try to use instead of ignore_errors. 

My colleague pointed out the use of the fail_key attribute in the getent module, by default it is set to YES which is why I chose to use the **ignore_errors: yes** was used in the getent task.   Instead use the getent module parameter **fail_key: no** instead.  This will create a variable ansible_facts.getent_passwd[pentest] with a null value for the dictionary.  This is why I said make sure you read all the available parameters and try to understand what they do.  I totally bypassed this parameter and didn't pay attention to it when I was reading the getent module documentation.  Sometimes the answer really is to RTFM! (Read The FULL Manpage)

```
---
- name: update users
  hosts: all
  gather_facts: false
  become: yes

  vars:
    s_user: 'pentest'
    s_expires: "{{ lookup('pipe','date +%s -d now+6months') }}"

  tasks:
  - name: GETENT {{s_user}} info
    getent:
      database: passwd
      key: "{{ s_user }}"
    fail_key: no

  - name: Change expiry date for user to today + 6 months
    user:
      name: "{{ s_user }}"
      # expires: -1
      expires: "{{ s_expires }}"
    when: (ansible_facts.getent_passwd[s_user] | type_debug) != 'NoneType'
```

The user task only updates the user when the getent_passwd variable for our user is is NOT a 'NoneType' or null.

So what was our approach to success:

1. Define our problem, now what we wanted to acheive and all the characteristics we could think of to be successul.
2. Solve one issue at a time.  Be iterative, eat the elephant one bite at a time.
3. You will encounter issues along the way that will take you will not forsee.
4. Keep checking against our criteria did we acheive success.
5. Ask for a multiple viewpoints, is there  better or just different ways to succeed.

There we did it we solved a problem that we need to solve and even with all the playing it is much faster than trying to update an expired account on 400+ servers manually.   And we can reuse this at a later time.

Bonus: What would you do to expire the account after three months if you no longer need to have this account active on your 400+ servers?

