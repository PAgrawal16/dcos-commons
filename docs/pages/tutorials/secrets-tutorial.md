---
title: Secrets Tutorial
---
# Secrets Tutorial

`dcos-commons` enables integration of DC/OS secrets both in declarative YAML API and flexible JAVA API. In YAML, secrets are declared within the `secret:` section in a pod specification. Similarly, in JAVA API, a `SecretSpec` is added to the `PodSpec` object. 

Please refer to [Developer Guide](developer-guide.html)  for more information about the JAVA API. Please refer to [Operations Guide](operations-guide.html) for detailed explaination of how DC/OS secrets can be used in dcos-commons. 

In this tutorial, we will use the existing `hello-world` service to experiment with secrets. First of all, create a DC/OS Enterprise 1.10 cluster (at least 3 nodes is recommended). Please note that integration of secrets in dcos-commons is only supported in DC/OS Enterprise 1.10 onwards.

## Create Secrets

Use DCOS CLI to create a secret.  You must have the Enterprise DC/OS CLI installed.

Install DC/OS Enterprise CLI:
```
$ dcos package install --cli dcos-enterprise-cli
```

Create a secret with path `hello-world/secret1`:
```
$ dcos security secrets  create -v "the value of secret1" hello-world/secret1
```

Now, lets create an RSA key and add its private key to a secret with path `hello-world/secret2` and its public key to another secret with path `hello-world/secret3`.

Create RSA key:
```
$openssl genrsa -out privatekey.pem 1024
$openssl req -new -x509 -key privatekey.pem -out publickey.cer -days 1825
```

Create secrets with value from these private and public key files:
```
$ dcos security secrets  create -f privatekey.pem  hello-world/secret2
```
```
$ dcos security secrets  create -f publickey.cer  hello-world/secret3
```


You can list secrets and view the content of a secret as follow:
```
$ dcos security secrets list  hello-world
```
```
$ dcos security secrets get hello-world/secret3
```


## Install Service with Secrets


The  `hello-world` package includes a sample YAML file for secrets. Please examine the following `examples/secrets.yml` file to grasp how secrets are declared:

````
name: {{FRAMEWORK_NAME}}
scheduler:
  principal: {{SERVICE_PRINCIPAL}}
  user: {{SERVICE_USER}}
pods:
  hello:
    container:
      image-name: ubuntu:14.04
    count: {{HELLO_COUNT}}
    placement: {{HELLO_PLACEMENT}}
    secrets:
      s1:
        secret: {{HELLO_SECRET1}}
        env-key: HELLO_SECRET1_ENV
        file: HELLO_SECRET1_FILE
      s2:
        secret: {{HELLO_SECRET2}}
        file: HELLO_SECRET2_FILE
    tasks:
      server:
        goal: RUNNING
        cmd: >
               env &&
               ls -la &&
               echo "secret files content" &&
               cat HELLO_SECRET*_FILE && echo &&
        ................
  world:
    count: {{WORLD_COUNT}}
    placement: {{WORLD_PLACEMENT}}
    secrets:
      s1:
        secret: {{WORLD_SECRET1}}
        env-key: WORLD_SECRET1_ENV
      s2:
        secret: {{WORLD_SECRET2}}
        file: WORLD_SECRET2_FILE
      s3:
        secret: {{WORLD_SECRET3}}
    tasks:
      server:
        goal: RUNNING
        cmd: >
               env &&
               ls -la &&
               echo "secret files content:" &&
               cat WORLD_SECRET*_FILE && echo &&
        ................
 ````


The `hello` pod has two secrets. The first secret with path `hello-world/secret1` is exposed both as an environment variable and as a file. The second one is exposed only as a file. The value of the second secret with path `hello-world/secret2`, that is the RSA private key, will be copied to the `HELLO_SECRET2_FILE` file located in the sandbox. 
  
 The `world` pod has three secrets. The first one is exposed only as an environment variable. The second and third secrets are exposed only as files. All `server` tasks in the `world` pod will have access to the value of these three secrets, either as a file and/or as an evironment variable.  Please note that the secret path is the default file path if no `file` keyword is given. Therefore, the file path for the third secret is same as the secret path.


Next, install `hello-world` package using the following "`option.json`" file:


```
$ dcos package install --options=option.json hello-world

```

```
$ cat option.json
{
      "service":{
          "spec_file" : "examples/secrets.yml"
      },
      "hello":{
          "secret1": "hello-world/secret1",
          "secret2": "hello-world/secret2"
      },
      "world":{
          "secret1": "hello-world/secret1",
          "secret2": "hello-world/secret2",
          "secret3": "hello-world/secret3"
          }
}
```

Here, we use `examples/secrets.yml` spec file. And, we also overwrite a couple of `hello` and `world` options -  `secret1`, `secret2`, and `secret3` parameters are set to specific secret paths.  Please examine `universe/config.json` for more information about the `hello-world` package configuration.

## Verify Secrets

You can use `dcos task exec` command to attach to the container that is running the task you want to examine. 

Please note that tasks of the `hello` pod in our example are running inside a docker image (`ubuntu:14.04`).

Run the following command to attach to the same container that is running the first task of the `hello` pod, that is `hello-0-server` (tasks name is `server`):

```
$ dcos task exec -it hello-0-server bash

> $ echo $HELLO_SECRET1_ENV
the value of secret1

> $ ls
HELLO_SECRET1_FILE  executor.zip	   stdout
HELLO_SECRET2_FILE  hello-container-path   stdout.logrotate.conf
containers	    stderr		   stdout.logrotate.state
executor	    stderr.logrotate.conf

$ cat HELLO_SECRET1_FILE 
the value of secret1

> $ cat HELLO_SECRET2_FILE 
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQC6Zpgb/CpM12ugwZXhuooOf8UzEkSqxHKMV50pTprrWgpVjoVL
y05hdHGQShppYuV/2HaluKUrvw4vwtfhaDpg8uQG27TVzYB9lgaglaJWwcUQoZGN
7/0g7EuK92z6aUyJSKt62aELCAmdPVEq5TgT8STUTuZ81hvDOG95FymNBwIDAQAB
AoGADUHArbTYeVCU2gEKnNw8d12E8+XntlF0aCDPD6IEiJqFw6H4PvS9pVa3wPBU
QoyDD/2gKpcgQCU9aA4udlyIUj+/zSSTeTgv/zT0AaqUNRpllf9ugUpcKye4LJWv
fy18fGRlsYYZFhjXwaD9yUUsP1G6Hu18dv8gf6yNDNgMYFkCQQDfGYWqbOZC0jea
kKkhRp5g0ft0CvASJrxNJgUT7FL5sUPDDmCWNYwfNIyq6+2XcG0FBZ/XRw+wto8r
b46OADezAkEA1eOcW+xHcEAhIZD0guuHPl1Ws3UceTEZU8XiRz1hPoXpshjnvrC+
ddRvRj+MHy0RwCidqwKJkGooS1p2wDzrXQJBAJ8pkx2x0VhMpxSjLbYqrmT+iXkR
MJKShfY4MJk1GUE/wMsQn8Gp9AxzLgPmiztmHrDdgVpRPRViOKPRU49lAlcCQA/F
7kT1IruLbyYLi4yQE/Qsa/VmAIiLb2O3Jx270A0NUROaNJTicdk8pkwW6Z1u9G0o
UaBH2p80xO3xqOo6U90CQQCXb34WCClMRQxl7fhp7n8UhwZpiSw74ZgcfclPmXN7
65eowN/M0F5cKed53mCUfAH4I81ShwucXoBF5F8xHDkR
-----END RSA PRIVATE KEY-----
````

Similarly, you can verify the content of secrets defined for any task of the `world` pod as follows:
```
$ dcos task exec -it world-0-server bash
 
> $ echo $WORLD_SECRET1_ENV
the value of secret1 

> $ ls
WORLD_SECRET2_FILE  executor.zip  stderr.logrotate.conf  stdout.logrotate.state
containers	    hello-world   stdout		 world-container-path1
executor	    stderr	  stdout.logrotate.conf  world-container-path2

> $ cat WORLD_SECRET2_FILE  
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQC6Zpgb/CpM12ugwZXhuooOf8UzEkSqxHKMV50pTprrWgpVjoVL
y05hdHGQShppYuV/2HaluKUrvw4vwtfhaDpg8uQG27TVzYB9lgaglaJWwcUQoZGN
7/0g7EuK92z6aUyJSKt62aELCAmdPVEq5TgT8STUTuZ81hvDOG95FymNBwIDAQAB
AoGADUHArbTYeVCU2gEKnNw8d12E8+XntlF0aCDPD6IEiJqFw6H4PvS9pVa3wPBU
QoyDD/2gKpcgQCU9aA4udlyIUj+/zSSTeTgv/zT0AaqUNRpllf9ugUpcKye4LJWv
fy18fGRlsYYZFhjXwaD9yUUsP1G6Hu18dv8gf6yNDNgMYFkCQQDfGYWqbOZC0jea
kKkhRp5g0ft0CvASJrxNJgUT7FL5sUPDDmCWNYwfNIyq6+2XcG0FBZ/XRw+wto8r
b46OADezAkEA1eOcW+xHcEAhIZD0guuHPl1Ws3UceTEZU8XiRz1hPoXpshjnvrC+
ddRvRj+MHy0RwCidqwKJkGooS1p2wDzrXQJBAJ8pkx2x0VhMpxSjLbYqrmT+iXkR
MJKShfY4MJk1GUE/wMsQn8Gp9AxzLgPmiztmHrDdgVpRPRViOKPRU49lAlcCQA/F
7kT1IruLbyYLi4yQE/Qsa/VmAIiLb2O3Jx270A0NUROaNJTicdk8pkwW6Z1u9G0o
UaBH2p80xO3xqOo6U90CQQCXb34WCClMRQxl7fhp7n8UhwZpiSw74ZgcfclPmXN7
65eowN/M0F5cKed53mCUfAH4I81ShwucXoBF5F8xHDkR
-----END RSA PRIVATE KEY-----

> $ ls hello-world/
secret3
> $ cat hello-world/secret3 
-----BEGIN CERTIFICATE-----
MIICsDCCAhmgAwIBAgIJAOqwzmYnPmezMA0GCSqGSIb3DQEBBQUAMEUxCzAJBgNV
BAYTAkFVMRMwEQYDVQQIEwpTb21lLVN0YXRlMSEwHwYDVQQKExhJbnRlcm5ldCBX
aWRnaXRzIFB0eSBMdGQwHhcNMTcwODEwMjEwNjE0WhcNMjIwODA5MjEwNjE0WjBF
MQswCQYDVQQGEwJBVTETMBEGA1UECBMKU29tZS1TdGF0ZTEhMB8GA1UEChMYSW50
ZXJuZXQgV2lkZ2l0cyBQdHkgTHRkMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKB
gQC6Zpgb/CpM12ugwZXhuooOf8UzEkSqxHKMV50pTprrWgpVjoVLy05hdHGQShpp
YuV/2HaluKUrvw4vwtfhaDpg8uQG27TVzYB9lgaglaJWwcUQoZGN7/0g7EuK92z6
aUyJSKt62aELCAmdPVEq5TgT8STUTuZ81hvDOG95FymNBwIDAQABo4GnMIGkMB0G
A1UdDgQWBBRHOUyzdlje5WyOwO9zqXyM8vDCuTB1BgNVHSMEbjBsgBRHOUyzdlje
5WyOwO9zqXyM8vDCuaFJpEcwRTELMAkGA1UEBhMCQVUxEzARBgNVBAgTClNvbWUt
U3RhdGUxITAfBgNVBAoTGEludGVybmV0IFdpZGdpdHMgUHR5IEx0ZIIJAOqwzmYn
PmezMAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEFBQADgYEAMUcYrG0SNZC+eLOG
bZi51mqaT/jmh42tmdTShg4EfhOj7g1mKmxFvPGZRC931ujN8yH3eguqCcLgrGiJ
td3ilFnBbT4Vyf5956vLjVZqyLMciwgMAJqqDhdIwHqh9ky1J8mQBpL9HtjmQDYl
xB7ZyP8k6rPqQAxI6LEqqQpzLkI=
-----END CERTIFICATE-----
```

Please note that the third secret for the `word` pod does not have a specific `file` keyword. Therefore, its secret path `hello-world/secret3` is also used as the file path by default. As can be seen in the output, `hello-world` directory is created and content of the third secret is copied to the `secret3` file. 



# Update Secret Value

We can update the content of a secret as follows:

```
$ dcos security secrets update -v "This is the NEW value for secret1" hello-world/secret1
```


Secret value is securely copied from the secret store to a file and/or to an environment variable. A secret file is an in-memory file and it disappears when all tasks of the pod terminate. 

Since we updated the value of `hello-world/secret1`,  we need to restart all associated pods to copy the new value from the secret store.


Lets restart the first `hello` pod. All tasks (there is only one task called `server` in our example) in the pod will be restarted.
```
$ dcos hello-world pod restart hello-0
```

As can be seen in the output, the `hello-0-server` task is already updated after the restart.  
```$ dcos task exec -it hello-0-server bash

> $ echo $HELLO_SECRET1_ENV                  
This is the NEW value for secret1

> $ cat HELLO_SECRET1_FILE 
This is the NEW value for secret1
```

Since we did not restart the `world` pods, their tasks still have the old value.  Run the following commands to examine the `world-0-server` task:
```
$ dcos task exec -it world-0-server bash
 
> $ echo $WORLD_SECRET1_ENV
the value of secret1 
```


Now, restart all remaining pods in order to get the new value of `hello-world/secret1`:
```$ dcos hello-world pod list
[
  "hello-0",
  "world-0",
  "world-1"
]

$ dcos hello-world pod restart world-0
$ dcos hello-world pod restart world-1
```

