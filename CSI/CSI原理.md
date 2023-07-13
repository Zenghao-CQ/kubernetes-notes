# CSI 架构

storageClass中，provisioner根据PVC创建PV即可，out-tree的pv对象被external-provisioner所watch，这里仅附近provision这一阶段。后续还有attach（external-attacher调用）和mount（kubelet调用），分别由CSI的driver和node负责

## pod挂载volume

流程：
* volume controller管理各类卷，其中PersistenVolumeController负责维护PV的状态（是否bound）并绑定PV
* 调度后，kubelet在宿主机为pod创建volume目录```
/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>```
* 两阶段（在Kubelet发起CRI请求前，由X完成宿主机目录的处理）：
    * Attach：将PV对应的存储设备（如远程存储介质，事实上有状态、持久化的存储多为远程）挂载到宿主机，非必需（如NFS可跳过）
    * Mount：将存储设备格式化，并挂载到宿主机上pod对应的volume目录
* Kubelet将Volume目录通过CRI里的Mounts参数，传递给容器运行时，为Pod对应的容器挂载volume

控制逻辑：
* Attach：相应逻辑独立于Kubelet主控制循环（Kubelet Sync Loop），这是由CSI标准设计决定的（CSI架构见下）

    kube-controller-manager（运行在master） 的一部分 => Volume Controller 维护 => AttachDetachController控制循环  
* Mount：该控制循环，属于kubelet的一部分，VolumeManagerReconciler，**避免volume逻辑阻塞主控制循环**
    Kubelet 另起goroutine =>  VolumeManagerReconciler

## CSI原理


## volume挂载流程
挂载存储卷时需经历三个的阶段：
* Provision/Delete（创盘/删盘）
* Attach/Detach（挂接/摘除）
* Mount/Unmount（挂载/卸载）

## CSI 标准
CSI标准[https://github.com/container-storage-interface/spec] 

CSI K8s实现文档[https://kubernetes-csi.github.io/docs/]

官方provisioner lib[https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner]