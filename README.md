# Install MariaDB Galera Cluster in Kubernetes Cluster

> [中文說明](./docs/zh-TW.md)

This repository will show you how to install MariaDB Galera cluster in Kubernetes cluster.

## Prepare for installation

You need to prepare persistent voluem for your database cluster. To do this, follow the steps below:

1. Create directory by issuing command below

    > This guide using home directory as persistent volume target folder, you can change path to what you want.

    ```console
    $ mkdir -p ~/.production/mariadb-galera
    ```

2. Make folder for every mariadb node.

    ```console
    $ cd mariadb-galera
    $ mkdir node1 node2 node3
    ```

3. Change the owner of directory to `1001`

    > We only change the owner of `mariadb-galera` folder but the `.production` folder.

    ```console
    # chown -R 1001:1001 mariadb-galera/
    ```

## Install

1. Change kubernetes yaml contents and apply it to kubernetes cluster

    > Don't forget to change folder path in yaml file.

    ```console
    $ cp deployment.yaml deployment.production.yaml
    $ vim deployment.production.yaml
    $ kubectl apply -f deployment.production.yaml
    ```

2. Apply SELinux policy

    ```console
    # semodule -i allowregistrypolicy.pp
    ```

    > If you create your folder in other directory but not your home directory, you need to change mariadbgalerapolicy.te to meet your situation.

    1. Your SELinux type will be shown after you issue `ls <YOUR_PATH> -alZ` command, the result will be shown between owner and size.
    2. Modify `mariadbgalerapolicy.te` file, add the type to `require` section.
    3. Modify `container_t` section below, change `user_home_t` into your type.
    4. Issue the commands below:

        > If you failed to apply these command, check your te file is using `LF` but `CRLF`.

        ```console
        # checkmodule -M -m -o mariadbgalerapolicy.mod mariadbgalerapolicy.te
        # semodule_package -o mariadbgalerapolicy.pp -m mariadbgalerapolicy.mod
        # semodule -i mariadbgalerapolicy.pp
        ```

3. Install MariaDB Galera by using Helm charts created by Bitnami

    > We install this charts to `database-system` namespace in Kubernetes, you can change it to what you want.

    > Change `<YOUR_ROOT_PASSWORD>` to what you want.

    ```console
    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    $ helm install mariadb-galera bitnami/mariadb-galera \
        --namespace database-system --create-namespace \
        --set rootUser.password=<YOUR_ROOT_PASSWORD>
    ```

4. Waiting for all nodes up and done. You can connect your database by port forward the 3306 port in Kubernetes cluster service.

## Update or Restart cluster

If you don't follow the steps below, the cluster will not be able to start. So please follow the steps carefully.

> Before you perform these actions, please **MAKE SURE YOU STILL REMEMBER THE ROOT PASSWORD AND BACKUP PASSWORD**, if you have forgotten it, please rescue these passwords or the clusters will be not able to start forever.

> You should fill the root password and backup password properly, if anyone have no password, the clusters are not able to start.

1. Remove the Helm chart of MariaDB Galera.
2. Backup for all of these clusters data folder.
    > This is an important step when these steps could lead data loss.

    > You can backup these folder in any methods, the easiest way is copy these folders to another path.
3. Reboot the system.
4. Wait for all other pods up, you can issue the commands below to restart your database cluster.

    > `<YOUR_ORIGINAL_ROOT_PASSWORD>` is your original root password and `<YOUR_ORIGINAL_BACKUP_PASSWORD>` is your original backup password.

    > You can specify which node to start by changing `<NODE_NUMBER>` value. The first node is 0 and so on.

    > The target namespace of the command is `database-system`, if this is not where you installed for your database cluster pods, you can change it to your namespace name.

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

5. Wait for all pods are up, issue the command below to remove the force start.

    ```console
    $ helm upgrade my-release my-repo/mariadb-galera \
        --set rootUser.password=<YOUR_ORIGINAL_ROOT_PASSWORD> \
        --set galera.mariabackup.password=<YOUR_ORIGINAL_BACKUP_PASSWORD> \
        --set podManagementPolicy=Parallel \
        --namespace database-system \
        --create-namespace
    ```

6. Done.

## Reference

- [MariaDB Galera packaged by Bitnami](https://github.com/bitnami/charts/tree/main/bitnami/mariadb-galera)
