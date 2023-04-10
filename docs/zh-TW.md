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
    # semodule -i allowmariadbpolicy.pp
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

    > 修改 `<YOUR_ROOT_PASSWORD>` 為你想要的 root 密碼

    ```console
    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    $ helm install mariadb-galera bitnami/mariadb-galera \
        --namespace database-system --create-namespace \
        --set rootUser.password=<YOUR_ROOT_PASSWORD>
    ```

4. 等待所有的節點起來後，你就可以 Port forward MariaDB Galera service 後進行連線

## 參考資料

- [MariaDB Galera packaged by Bitnami](https://github.com/bitnami/charts/tree/main/bitnami/mariadb-galera)
