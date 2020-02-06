Container Secrets Patterns
===

### 版控存放
將 secret 透過版本控管以加密檔案存儲方式

### context
要將軟體發佈至營運的環境中時, 不可避免的一定會有許多參數需要進行設定。而其中有些機敏性的參數(在此我們稱為 secret)，比如資料庫密碼，外部 ftp 的帳號密碼，API Key 等等。這些有高度資安疑慮不可外洩的資訊要以一個安全可追蹤的方式來進行儲存且發佈到軟體運行的環境。

### problem
如何安全存取 secret

#### force
- secret 不可採用明碼方式儲存
- 專案結構較為簡單，只使用到一組 secret，不會被許多不同的服務或專案存取
- 在部署系統時若secret可隨程式模組一起發佈並於本地端直接存取，則可簡化持續整合/持續發布的流程與系統架構。

#### solution

將 secret 存放在版本控管的系統中，利用版本控管支援 hook script 的功能，在將檔案存入主要的 repository 時觸發 hook script, 將所指定的檔案進行加密。反之，在取得檔案時，要將所指定的檔案解密。
![fig. 1 版控存放](https://i.imgur.com/gzflm9E.png)

**以 [Git-crypt](https://github.com/AGWA/git-crypt) 為例**

範例會先建立一個空的 git repository, clone 出來之後，進行 git-crypt 的設定。再觀察對應檔案加解密的狀況。

```bash
# 1. 建立空的 repository
$ cd /your/folder/
$ mkdir remote-repo
$ cd remote-repo; git init --bare
$ git clone ./remote-repo client-with-git-crypt

# 2. 在 clone 出來的 repository 中設定 git-crypt
$ cd client-with-git-crypt
$ echo "secretfile filter=git-crypt diff=git-crypt" > .gitattributes
## 這裡可以設定要加密的檔案, 比對到的檔案會自動進行加解密
$ echo "*.key filter=git-crypt diff=git-crypt" >> .gitattributes
$ git-crypt init
$ git-crypt export ../key-of-git-crypt #將 key 匯出來

# 3. 新增檔案，一個會被加密，另一個不會
echo "It will be encrypted" > aa.key
echo "It will not be encrypted" > bb.txt
$ git add *; git commit -m "initial commit"
$ git push

# 4. 檢查是否真的加密
$ git clone ./remote-repo client-without-git-crypt
$ cd client-without-git-crypt
## 檔案內容已經成為密文
$ cat aa.key
GITCRYPT
nf���C�8�0��������S�Q㜦�S�AE%
## 不符合比對設定的檔案仍然是明文
$ cat bb.txt
It will not be encrypted

# 開啟自動解密功能
$ git-crypt unlock ../key-of-git-crypt

# 解除自動解密功能
$ git-crypt lock ../key-of-git-crypt
```

#### result context
- 除了原來有在使用版本控管的服務之外不需要再另外加其他的服務便可以運行
- 在多人協作的環境之下需要將相關的加密 key 存放在 使用者的設備上，對於在使用者設備上的加密 key 控管難度較高
- 當專案複雜度變高，新增多種不同服務時。必需將 secret 同步至不同的 repository 中，會增加管理上的複雜度。

#### known used

[Git-crypt](https://github.com/AGWA/git-crypt)

[BlackBox](https://github.com/StackExchange/blackbox)

[SOPS](https://github.com/mozilla/sops)

[Transcrypt](https://github.com/elasticdog/transcrypt)

#### REFERENCE
[Secret Management Architectures: Finding the balance between security and complexity](https://medium.com/slalom-technology/secret-management-architectures-finding-the-balance-between-security-and-complexity-9e56f2078e54)

[Securing data in your repository with git-crypt – an android example](https://www.schibsted.pl/blog/securing-data-with-git-crypt/)

[4 secrets management tools for Git encryption](https://opensource.com/article/19/2/secrets-management-tools-git)

-------
### 中央存放
將 secret 以中央服務(secret management service)方式存儲

### problem
如何安全存取 secret

### context
要將軟體發佈至營運的環境中時。不可避免的一定會有許多參數需要進行設定。而其中有些機敏性的參數(在此我們稱為 secret)，比如資料庫密碼，外部 ftp 的帳號密碼，API Key 等等。這些有高度資安疑慮不可外洩的資訊要以一個安全可追蹤的方式來進行儲存且發佈到軟體運行的環境。

#### force 
- secret 不可採用明碼方式儲存
- 專案的結構比較複雜，有多個不同的服務要存取同一組 secret ，或是多個服務使用多組secret。
- 多個secret如果分開儲存，會造成管理的困難甚至造成安全疑慮。例如，需要設定不同軟體模組讀取不同secret的權限。

#### solution
![fig. 2 中央存放 - 發佈控制](https://i.imgur.com/m5pq8Uy.png)


將 secret 存放在 secret management service 中。secret management service 以服務的方式存在於系統中，並提供權限控管，可依 secret 來定義存取的權限。 secret 管理者利用 secret management service 所提供的工具進行 secret 的管理。

以 [K8S Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/#%e6%9c%80%e4%bd%b3%e5%ae%9e%e8%b7%b5) 為例

**建立 Secret**
```shell=
# Create files needed for rest of example.
$ echo -n 'ftpadmin' > ./ftp-username
$ echo -n 'password-of-ftp' > ./ftp-password
$ echo -n 'dbadm' > ./db-username
$ echo -n 'password-of-db' > ./db-password

# 以匯入檔案的方式建立 secret
$ kubectl create secret generic ftp-user-pass --from-file=./ftp-username --from-file=./ftp-password
secret "ftp-user-pass" created

$ kubectl create secret generic db-user-pass --from-file=./db-username --from-file=./db-password
secret "db-user-pass" created

# 取得 secret
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      7s
ftp-user-pass         Opaque                                2      32s

# 取得 secret 的詳細資料
$ kubectl describe secrets/db-user-pass
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
db-password:  14 bytes
db-username:  6 bytes
```

**存取控管 - 將 secret 佈署至 pod 中**

將 pod 可以使用的參數設定到 pod 的  enviroment variable 中

這裡是一個以  enviroment variable 進行 secret 配置的設定，檔名為 `secret-envars-pod.yaml`
```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: secret-envars-test-pod
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-user-pass
          key: db-username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-user-pass
          key: db-password
```

啟動 pod，然後進入 pod 中檢查是否真的可以存取 secret
```bash=
$ kubectl exec -it secret-envars-test-pod -- /bin/bash

root@secret-envars-test-pod:/ printenv
### 省略  
SECRET_PASSWORD=password-of-db
SECRET_USERNAME=dbadm
```
在該 pod 中只能取用所指定的 secret，以此方式來達到權限控管的目的


#### result context
- secret 可以被存放在一個集中管理的服務中
- secret 的存取可以被有效的管理及限制，對於權限也可以讓管理人員方便的管理
- 需要使用額外的設備來提供 secret management service 的服務
- 會增加服務設定的的複雜度

#### known used
- Hashicorp Vault
- CyberArk Conjur
- AWS Secrets Manager
- k8s secret

#### Reference
[distribute-credentials-secure](https://kubernetes.io/zh/docs/tasks/inject-data-application/distribute-credentials-secure/)

[HashiCorp Secrets Engines](https://learn.hashicorp.com/vault/getting-started/secrets-engines)

## 發佈模式 -- (還未完成)

### context

已經設定好的 secret 需要被發佈到程式所運行的環境中，這樣子才可以被應用程式使用。而發佈的方式有常見的幾種

1. 利用直接讀取 secret management
2. 設定在環境變數中
3. 利用 volumn mount 讀取 file
4. config file templat replacement

### Problem
我們如何能夠讓受管理的 secret 被安全的發佈至環境中並合理的被使用

### - secret as Enviroment Vairable
#### force

#### solution
管理人員透過設定 Key/Value 的方式將對應的 secret 設定至 `secret management service` 中，對應的`Template Replacement Service` 接收到資料變更的通知時會即時將已變更的 secret 即時設到環境中。

#### result context
- 

#### known as 
k8s + vault + [vault-env](https://github.com/channable/vaultenv)

### - secret 管理 Template 套版發佈模式
---
#### force
- 不想將 secret 被放至在 enviroment variable 的時後


#### solution
`Template Replacement` 程式在得知 secret 變更時會即時將已變更的 secret 代入 設定檔 Template 中之後將置換完成的設定檔佈署至環境。應用程式以直接讀取經過處理的設定檔來取用 secret。

以 Vault with Consul-Template 為例，
```
production:{{with $secret := vault "secret/my-app/production" }}
    adapter: postgresql
    host: {{key "my-app/production/host"}}
    username: {{$secret.Data.username}}
    password: {{$secret.Data.password}}
{{end}}
```
Consul-Template 會把 雙大括弧 中的內容取代為 vault 中對應的值。之後利用 CI/CD 的過程將 config 發佈至環境中。

#### resulting context
- secret 

#### known as 
[confd](https://github.com/kelseyhightower/confd)
[Consul/Consul-Template](https://github.com/hashicorp/consul-template)

#### REF
[Using vault with consul tempalte](https://www.hashicorp.com/blog/using-vault-with-consul-template/)

### - Use volume mounts 
---
#### force
- 使用 container 技術於運行環境
- 不想把 secret 放置在環境變數中
- 應用程式對於 secret 只有讀取的能力
- 想導入 DevOps 的流程

#### solution
管理人員透過設定 Key/Value 的方式將對應的 secret 設定至 `secret management service`, 並利用 volumn mount 的方式讓 container 得到讀取的權限。應用程式利用 readfile 的方式來對 secret 進行讀取。對應的 volumn driver 會將所需要的 secret 回傳。

- [K8S secret Volumn Mount](https://kubernetes.io/zh/docs/tasks/inject-data-application/distribute-credentials-secure/#%E5%88%9B%E5%BB%BA%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E5%8D%B7%E8%AE%BF%E9%97%AE-secret-%E6%95%B0%E6%8D%AE%E7%9A%84-pod)
```
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
          # name must match the volume name below
          - name: secret-volume
            mountPath: /etc/secret-volume
  # The secret data is exposed to Containers in the Pod through a Volume.
  volumes:
    - name: secret-volume
      secret:
        secretName: test-secret
```

### result context
1. secret 發佈的過程可以自動化的進行
2. 可以將 secret 只發佈有權限的應用程式，對 secret 的取用進行控管
3. 應用程式對 secret 只有讀取的權限，無法任意修改 secret



#### known as 
docker + vault + Volume Driver - LibSecret
k8s secret volumn mount

#### REF
https://www.katacoda.com/courses/docker-production/docker-volume-libsecret#
> how to step by step

https://banzaicloud.com/blog/inject-secrets-into-pods-vault-revisited/
> in memory volumn

## reference
[k8s configmap](https://medium.com/@vincentkernn/deploying-kubernetes-with-configmap-and-secrets-5aff29077b8d)

[k8s secret](https://kubernetes.io/docs/concepts/configuration/secret/)

[secret management](https://blog.cryptomove.com/secrets-management-guide-approaches-open-source-tools-commercial-products-challenges-db560fd0584d)
###### tags: `patterns`