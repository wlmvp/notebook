- 如何关联pvc到特定的pv?

  我们可以使用对 pv 打 label 的方式，具体如下：

  创建 pv，指定 label

  ```
  -[appuser@chenqiang-dev pvtest]$ cat nfs-pv2.yaml 
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: nfs-pv2
    namespace: chenqiang-pv-test
    labels:
      pv: nfs-pv2
  spec:
    capacity:
      storage: 100Mi
    accessModes:
  
     - ReadWriteMany
       nfs:
  
      # FIXME: use the right IP
  
  ​    server: 10.130.44.20
  ​    path: "/test/mysql-nfs01"
  ```

  然后创建 pvc，使用 matchLabel 来关联刚创建的 pv:nfs-pv2

  ```
  -[appuser@chenqiang-dev pvtest]$ cat nfs-pvc2.yaml   
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nfs-pvc2
    namespace: chenqiang-pv-test
  spec:
    accessModes:
      - ReadWriteMany
    storageClassName: ""
    resources:
      requests:
        storage: 90Mi
    selector:
      matchLabels:
        pv: nfs-pv2
  ```

  
  下面开始测试： 
  先创建3个pv

  下面开始测试： 
  先创建3个pv

  ```
  -[appuser@chenqiang-dev pvtest]$ kubectl get pv
  NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                        STORAGECLASS         REASON    AGE
  nfs-pv1                                    100Mi      RWX            Retain           Bound       chenqiang-pv-test/nfs-pvc1                                  41m
  nfs-pv2                                    100Mi      RWX            Retain           Available                                                               2m
  nfs-pv3                                    100Mi      RWX            Retain           Available                                                               2m
  nfs-pv4                                    100Mi      RWX            Retain           Available                                                               2m
  nfs-server-pv                              100Gi      RWX            Retain           Bound       default/nfs-server-pvc  
  
  ```


  然后创建 pvc

  ```
  -[appuser@chenqiang-dev pvtest]$ kubectl apply -f nfs-pvc2.yaml 
  persistentvolumeclaim "nfs-pvc2" created
  -[appuser@chenqiang-dev pvtest]$ kubectl apply -f nfs-pvc3.yaml 
  persistentvolumeclaim "nfs-pvc3" created
  -[appuser@chenqiang-dev pvtest]$ kubectl apply -f nfs-pvc4.yaml 
  persistentvolumeclaim "nfs-pvc4" created
  ```

  
  看看 pvc是否创建成功，及是否正确绑定到特定的 pv，执行如下命令：

  看看 pvc是否创建成功，及是否正确绑定到特定的 pv，执行如下命令：

  ```bash
  -[appuser@chenqiang-dev pvtest]$ kubectl -n chenqiang-pv-test get pvc
  NAME       STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  nfs-pvc1   Bound     nfs-pv1   100Mi      RWX                           41m
  nfs-pvc2   Bound     nfs-pv2   100Mi      RWX                           25s
  nfs-pvc3   Bound     nfs-pv3   100Mi      RWX                           17s
  nfs-pvc4   Bound     nfs-pv4   100Mi      RWX                           10s
  -[appuser@chenqiang-dev pvtest]$ kubectl get pv
  NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                        STORAGECLASS         REASON    AGE
  nfs-pv1                                    100Mi      RWX            Retain           Bound      chenqiang-pv-test/nfs-pvc1                                  42m
  nfs-pv2                                    100Mi      RWX            Retain           Bound      chenqiang-pv-test/nfs-pvc2                                  4m
  nfs-pv3                                    100Mi      RWX            Retain           Bound      chenqiang-pv-test/nfs-pvc3                                  4m
  nfs-pv4                                    100Mi      RWX            Retain           Bound      chenqiang-pv-test/nfs-pvc4                                  4m
  
  ```


  完美！都正确绑定了。
  
