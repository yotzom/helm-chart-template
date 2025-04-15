# 安裝說明
## 目錄結構
### helm-templateproject
│── Chart.yaml
│── charts    
│── values.yaml
│── env-values     <--  存放不同環境的個別參數檔, 例如prod, stage, alpha, test1, test2
│        │── alpha.yaml
│        │── prod.yaml
│        └── stage.yaml
│── files          <--  用來在Jenkins CI時掛載檔案放進secret用的, 需與git repo https-certs一起使用
│        └── secrets
│               └── placeholder
│── sub-services          <-- 存放不同子服務設定用, 新增服務只需在此多新增一份設定檔即可使用
│        │── sample-service.yaml
│        │── templateproject.yaml
│        │── ...
│        │── ..
│        │── admin.yaml
│        │── client.yaml
│        └── task-liquibase-executor.yaml
└── templates
           │── NOTES.txt
           │── _helpers.tpl
           │── configmap.yaml
           │── deployment.yaml
           │── frontend-nginx-configmap.yaml
           │── hpa.yaml
           │── ingress.yaml
           │── job.yaml
           │── promtail-configmap.yaml
           │── secrets.yaml
           │── service.yaml
           │── serviceaccount.yaml
           └── tests
                 └── test-connection.yaml
---
## 背景知識說明
#### 參數覆蓋
當你有使用到不同的參數檔時, 只要是相同yaml路徑的值就會被覆蓋,
範例: values.yaml中有app.replica=1, prod.yaml中有app.replica=3
在這種情況下這兩個參數會覆蓋, 參數覆蓋邏輯參考下方順序
#### 參數覆蓋順序
1. values.yaml：Chart 中的預設配置, 就算沒有特別指定也會預設吃入此yaml檔。
2. -f 或 --values 命令行參數：指定的外部 YAML 配置文件。
3. --set 命令行參數：通過命令行設置的參數。

---
## 參數調用邏輯說明
Jenkins的自動安裝指令中就會使用以下三個參數檔做安裝, 
`env-values/{env}.yaml`會覆蓋`values.yaml`,
`sub-service/{service_name}.yaml`不會載入進Values中, 
會額外載入至其他變數中做調用, 所以不會有覆蓋問題

#### 參數檔共會從下面三個地方載入:
- values.yaml
- env-values/{env}.yaml
- sub-service/{service_name}.yaml
#### 各參數檔用法說明
##### values.yaml
裡面放各服務都會用到的`通用預設值`
##### env-values/{env}.yaml
按照各`環境`區分不同的yaml檔存放, 裡面只包含`各環境會不同的設定`, 例如resource, ingress
##### sub-service/{service_name}.yaml
按照各`服務`區分不同的yaml檔存放, `每一個yaml都只包含自己服務的完整設定`

---
## 安裝方法
#### 因為需要掛載額外的httpscerts憑證, 所以只限使用Jenkins做CI&CD
Alpha Jenkins link: https://jenkins-alpha.xxx.com/job/templateproject/job/ALPHA-K8S-templateproject-Deploy/
Stage Jenkins link: https://jenkins-stage.xxx.com/job/templateproject/job/STAGE-K8S-templateproject-Deploy/
Prod Jenkins link: https://jenkins-prod.xxx.com/job/templateproject/job/templateproject-deploy/
#### 開發helm chart時debug用的安裝指令
```
helm upgrade -n {dev_namespace} templateproject . -f env-values/stage.yaml \
--set version=stage,replicaCount=1,serviceName=templateproject,image.tag=prod_v1.0_ca1751a0asc8b295aa5f89bz459d4f52g5j56121 \
--install --create-namespace --wait --wait-for-jobs --timeout 300s --force --debug --dry-run
```

---
## 特殊的業務邏輯

##### 1. https憑證掛載 
只有aaa和bbb這兩個服務才會創建secret並掛載
##### 2. job類型的服務 
task-liquibase-executor
##### 3. HPA啟用條件 
當.Values.version等於prod並且服務不是task-liquibase-executor和service-templateproject-cron-standalone,
因為service-templateproject-cron-standalone僅限存在1台
##### 4. log-agent promtail啟用條件
version=prod
---
## 新增新服務時修改方法
