# Install MariaDB Galera Cluster in Kubernetes Cluster

這份文件將會告訴你如何安裝 MariaDB Galera 至 Kubernetes 叢集中

## 準備安裝

你需要先準備 PV 給資料庫儲存，依據已下的指令進行 PV 建立

1. 建立資料夾

    > 這個教學使用家目錄當作 PV 的資料夾，你可以更改為你想要的路徑

    ```console
    $ mkdir -p ~/.production/mariadb-galera
    ```

2. 每個資料庫節點建立一個專用資料夾

    ```console
    $ cd mariadb-galera
    $ mkdir node1 node2 node3
    ```

3. 變更資料夾擁有者為 `1001`

    > 我們只變更 `mariadb-galera` 資料夾而非整個 `.production` 資料夾

    ```console
    # chown -R 1001:1001 mariadb-galera/
    ```

## 安裝

1. 修改 Kubernetes yaml 檔並套用之

    > 不要忘記變更 yaml 檔內的資料夾路徑

    ```console
    $ cp deployment.yaml deployment.production.yaml
    $ vim deployment.production.yaml
    $ kubectl apply -f deployment.production.yaml
    ```

2. 套用 SELinux 策略

    ```console
    # semodule -i mariadbgalerapolicy.pp
    ```

    > 若你非建立資料夾在家目錄下，請依據以下步驟修改 te 檔以符合你的需求

    1. 執行 `ls <YOUR_PATH> -alZ` 後，SELinux 的 type 會顯示於擁有者與大小之間
    2. 修改 `mariadbgalerapolicy.te` 檔案，將 1. 中的 type 新增到 `require` 區塊中
    3. 修改 `container_t` 區塊，將 `user_home_t` 修改為 1. 中的 type
    4. 執行以下指令

        > 若無法正常套用策略，請檢查 te 檔案的換行結尾使用 `LF` 而非 `CRLF`。

        ```console
        # checkmodule -M -m -o mariadbgalerapolicy.mod mariadbgalerapolicy.te
        # semodule_package -o mariadbgalerapolicy.pp -m mariadbgalerapolicy.mod
        # semodule -i mariadbgalerapolicy.pp
        ```

3. 使用 Bitnami 的 Helm charts 安裝 MariaDB Galera

    > 這將會安裝 MariaDB Galera 至 `database-system` 命名空間下，你可以安裝在其它的命名空間下

    > 修改 `<YOUR_ROOT_PASSWORD>` 與 `<YOUR_RECOVERY_PASSWORD>` 為你想要的密碼

    ```console
    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    $ helm install mariadb-galera bitnami/mariadb-galera \
        --namespace database-system --create-namespace \
        --set rootUser.password=<YOUR_ROOT_PASSWORD> \
        --set galera.mariabackup.password=<YOUR_RECOVERY_PASSWORD>
    ```

4. 等待所有的節點起來後，你就可以 Port forward MariaDB Galera service 後進行連線

## 升級版本或重新啟動

若沒有特別設定，重新啟動系統後會導致整個資料庫叢集無法啟動，所以務必確定有執行以下動作

> 執行以下動作前**請先確定是否仍記得目前的 root 密碼與備份密碼**，如忘記請先想辦法取回這兩組密碼，以免叢集無法正常啟動

> root 密碼與備份密碼都**不可以留空**，只要有一方的密碼未設定，這個資料庫叢集就沒救了

1. 移除 MariaDB Galera 的 Helm charts
2. 備份資料庫叢集的資料夾
    > 此步驟至關重要，特別是當這些資料庫已經在正式環境服務過後，這些資料更是不能遺失，請務必做好備份動作，以免所有的資料遺失

    > 備份方法不限，最簡單的方式就是直接複製到另外一個資料夾進行備份，畢竟重新啟動系統勢必是為了安裝系統更新，只要不會讓資料遺失的備份方式都 OK
3. 重新啟動系統
4. 待所有的 Pod 都啟動後以下面的指令重新啟動 MariaDB Galera 資料庫叢集

    > `<YOUR_ORIGINAL_ROOT_PASSWORD>` 填原本的 root 密碼，`<YOUR_ORIGINAL_BACKUP_PASSWORD>` 則填原本的備份密碼

    > `<NODE_NUMBER>` 可以自行決定要從哪台節點進行啟動，第一台節點為 0，以此類推

    > 此指令的目標命名空間為 `database-system`，若非安裝在此命名空間，請自行修改該值

    ```console
    $ helm install mariadb-galera bitnami/mariadb-galera \
        --set rootUser.password=<YOUR_ORIGINAL_ROOT_PASSWORD> \
        --set galera.mariabackup.password=<YOUR_ORIGINAL_BACKUP_PASSWORD> \
        --set galera.bootstrap.forceBootstrap=true \
        --set galera.bootstrap.bootstrapFromNode=<NODE_NUMBER> \
        --set podManagementPolicy=Parallel \
        --namespace database-system \
        --create-namespace
    ```

5. 待所有 Pod 都正常啟動後，執行以下的指令移除強制啟動

    ```console
    $ helm upgrade my-release my-repo/mariadb-galera \
        --set rootUser.password=<YOUR_ORIGINAL_ROOT_PASSWORD> \
        --set galera.mariabackup.password=<YOUR_ORIGINAL_BACKUP_PASSWORD> \
        --set podManagementPolicy=Parallel \
        --namespace database-system \
        --create-namespace
    ```

6. 完成

## 節點重啟後復原狀態

可能因為某些原因導致 K8s 的節點重啟，此時 MariaDB Galera 叢集必定起不來，必須依照下列步驟復原其狀態

> 在這個範例中我們假設 MariaDB Galera 叢集被安裝在 `mariadb-galera-clusters` 命名空間中

1. 取得所有 PVC 名稱

    ```console
    $ kubectl get pvc --namespace mariadb-galera-clusters
    ```

2. 執行以下指令以找出哪個 MaraiDB Galera 節點有 `safe_to_bootstrap=1` 設定值

    > 每個 PVC 名稱都要執行一次

    ```console
    kubectl run -i --rm --tty volpod --overrides='
    {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
            "name": "volpod"
        },
        "spec": {
            "containers": [{
                "command": [
                    "cat",
                    "/mnt/data/grastate.dat"
                ],
                "image": "bitnami/minideb",
                "name": "mycontainer",
                "volumeMounts": [{
                    "mountPath": "/mnt",
                    "name": "galeradata"
                }]
            }],
            "restartPolicy": "Never",
            "volumes": [{
                "name": "galeradata",
                "persistentVolumeClaim": {
                    "claimName": "<YOUR PVC NAME>"
                }
            }]
        }
    }' --image="bitnami/minideb" --namespace mariadb-galera-clusters
    ```

3. 這邊你會有兩種狀況:

    > 在這個範例中我們假設你的 Helm repo 名稱是 `bitnami` 且安裝的 chart 名稱為 `mariadb-galera`
    - 只有一個節點有 `safe_to_bootstrap=1` 設定值
        你可以利用該節點重啟整個叢集

        ```console
        $ helm install mariadb-galera bitnami/mariadb-galera \
            --namespace mariadb-galera-clusters \
            --set rootUser.password=<YOUR_DB_ROOT_PASSWORD> \
            --set galera.mariabackup.password=<YOUR_DB_BACKUP_PASSWORD> \
            --set galera.bootstrap.forceBootstrap=true \
            --set galera.bootstrap.bootstrapFromNode=<THE_NODE_NUMBER_YOU_GET_ABOVE> \
            --set podManagementPolicy=Parallel
        ```

    - 所有節點都為 `safe_to_bootstrap=0`
        你必須指定一個節點重啟整個叢集，在這個範例我們使用第 0 個節點，你可以自行選擇

        ```console
        $ helm install mariadb-galera bitnami/mariadb-galera \
            --namespace mariadb-galera-clusters \
            --set rootUser.password=<YOUR_DB_ROOT_PASSWORD> \
            --set galera.mariabackup.password=<YOUR_DB_BACKUP_PASSWORD> \
            --set galera.bootstrap.forceBootstrap=true \
            --set galera.bootstrap.bootstrapFromNode=0 \
            --set galera.bootstrap.forceSafeToBootstrap=true \
            --set podManagementPolicy=Parallel
        ```

4. 待所有的節點都起來後，必須將強制啟動的設定移除

    ```console
    helm upgrade mariadb-galera bitnami/mariadb-galera \
        --namespace mariadb-galera-clusters \
        --set rootUser.password=<YOUR_DB_ROOT_PASSWORD> \
        --set galera.mariabackup.password=<YOUR_DB_BACKUP_PASSWORD> \
        --set podManagementPolicy=Parallel
    ```

5. 完成

## 參考資料

- [MariaDB Galera packaged by Bitnami](https://github.com/bitnami/charts/tree/main/bitnami/mariadb-galera)
