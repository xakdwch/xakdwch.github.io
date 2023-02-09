# **Overview**

k8s是通过kube-apiserver组件将数据存储在etcd中，这些数据通常会通过protobuf或者json序列化后进行存储，而有的资源则需要进行加密存储，比如Secret。本文就以Secret资源为例，手把手教你如何解密K8s集群的加密资源。

# **被加密的资源**

k8s中有的资源在写入etcd之前，是会被进行加密存储的，最常见的就是secret资源。

我们可以通过kube-apiserver的manifests文件中的启动参数可以知道k8s有哪些资源会被加密，比如：

```yaml
apiVersion: v1
 kind: Pod
 metadata:
 annotations:
   kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.180.130:6443
 creationTimestamp: null
 labels:
   component: kube-apiserver
   tier: control-plane
 name: kube-apiserver
 namespace: kube-system
 spec:
 containers:
 - command:
   - kube-apiserver
   - --advertise-address=192.168.180.130
   - --allow-privileged=true
   ...
   - --encryption-provider-config=/etc/kubernetes/pki/secrets-encryption.yaml
   ...
```

从上述kube-apiserver的yaml文件可以看到，其中有一个--encryption-provider-config的配置项，该配置项对应的是一个yaml文件路径，而该文件中保存的正是k8s中被加密的资源信息。下面我们看一下该文件中的内容：

```yaml
kind: EncryptionConfig
 apiVersion: v1
 resources:
 - resources:
   - secrets
   providers:
   - aescbc:
       keys:
       - name: key
         secret: GPG4RC0Vyk7+Mz/niQPttxLIeL4HF96oRCcBRyKNpfM=
   - identity: {}
```

该yaml文件中记录了k8s中被加密的资源列表及对应的加密算法配置信息：

1. resources.resources中记录的是被加密的资源列表，可以看到只有secrets资源被加密了。
2. resources.providers中记录的是对应的加密算法配置信息，可以看到只有aescbc和identity这两种算法配置，其中identity为空，所以secret是通过AES-CBC加密的。除此之外，其实k8s还支持aesgcm、secretbox、kms这几种算法配置。

# **解析流程**

对于这类进行加密存储的k8s资源，我们通常需要进行以下流程来解码，才能最终获得资源的明文信息：

1. 根据key从etcd中获取被加密的value
2. 对被加密的value进行解密，得到被解密的value
3. 通过k8s的解码器对解密后的value进行解码，最终得到资源的明文信息

# **获取被加密的value**

首先，我们需要创建etcd客户端，然后根据资源在etcd中的key来获取value。k8s资源在etcd中的prefix默认为/registry/，如下所示：

```shell
 /registry/secrets/rook-ceph/rook-ceph-rgw-token-vd98q
 /registry/secrets/rook-ceph/rook-ceph-system-token-rnpms
 /registry/secrets/rook-ceph/rook-csi-cephfs-plugin-sa-token-n5qdq
 /registry/secrets/rook-ceph/rook-csi-cephfs-provisioner-sa-token-n7947
 /registry/secrets/rook-ceph/rook-csi-rbd-plugin-sa-token-qcknl
 /registry/secrets/rook-ceph/rook-csi-rbd-provisioner-sa-token-wrn6w
 /registry/serviceaccounts/default/default
 /registry/serviceaccounts/kube-node-lease/default
 /registry/serviceaccounts/kube-public/default
 /registry/serviceaccounts/kube-system/attachdetach-controller
 /registry/serviceaccounts/kube-system/bootstrap-signer
```

这里，我们获取/registry/secrets/rook-ceph/rook-ceph-rgw-token-vd98q这个key的value，而这个key对应的是k8s中rook-ceph这个命名空间下的一个名为rook-ceph-rgw-token-vd98q的secret资源。

# **创建etcd client**

```go
func newEtcdClient() (*clientv3.Client, error) {
  cfgTLS := &transport.TLSInfo{
    CertFile:      "/etc/kubernetes/pki/etcd/server.crt",
    KeyFile:       "/etc/kubernetes/pki/etcd/server.key",
    TrustedCAFile: "/etc/kubernetes/pki/etcd/ca.crt",
  }
 
  tlsCfg, err := cfgTLS.ClientConfig()
  if err != nil {
    return nil, err
  }
 
  client, err := clientv3.New(clientv3.Config{
    Endpoints:   []string{"192.168.180.130:2379"},
    DialTimeout: 2 * time.Second,
    TLS:         tlsCfg,
  })
  if err != nil {
    return nil, err
  }
 
  return client, nil
}
```

# **获取value**

调用etcd client的Get方法获取key对应的value：

```go
 etcdClient, err := newEtcdClient()
 if err != nil {
  t.Fatal(err)
 }
 
 resp, err := etcdClient.Get(context.TODO(), testSecretKey)
 if err != nil {
  t.Fatal(err)
 }
 encryptedData := resp.Kvs[0].Value
```

这样，我们就得到了被加密的value。

# **解密value**

解密value需要借助k8s中的Transformer来完成，Transformer提供了两个方法：把从etcd中读出的数据进行解密；把即将写入etcd中的数据进行加密。

我们需要按照以下流程对数据进行解密：

- 首先，我们要根据上述--encryption-provider-config配置项对应的yaml文件，初始化Transformer，每种被加密的resource都会有对应的Transformer：

```go
 transformers, err := encryptionconfig.GetTransformerOverrides("/etc/kubernetes/pki/secrets-encryption.yaml")
 if err != nil {
  t.Fatal(err)
 }
```

- 调用Transformer的TransformFromStorage方法来解密：

```go
 gr := schema.ParseGroupResource("secrets")
 transformer := transformers[gr]
 decryptedData, _, err := transformer.TransformFromStorage(encryptedData, authenticatedDataString(testSecretKey))
 if err != nil {
  t.Fatal(err)
 }
```

至此，我们就得到了经过解密的value了。

# **解码value**

k8s中的资源写入etcd前，通常会经过protobuf或json进行编码。因此，我们还需利用k8s的解码器对解密后的value进行解码。

利用解码器的Decode方法进行解码：

```go
 var secret v1.Secret
 if _, _, err := legacyscheme.Codecs.UniversalDeserializer().Decode(decryptedData, nil, &secret); err != nil {
  t.Fatal(err)
 }
 
 unst, err := runtime.DefaultUnstructuredConverter.ToUnstructured(&secret)
 if err != nil {
  t.Fatal(err)
 }
```

最后将secret转成map，主要是为了便于阅读。

# **完整代码**

上述示例的完整代码如下：

```go
 package test
 
 import (
 "context"
 "go.etcd.io/etcd/clientv3"
 "go.etcd.io/etcd/pkg/transport"
 v1 "k8s.io/api/core/v1"
 "k8s.io/apimachinery/pkg/runtime"
 "k8s.io/apimachinery/pkg/runtime/schema"
 "k8s.io/apiserver/pkg/server/options/encryptionconfig"
 "k8s.io/apiserver/pkg/storage/value"
 "k8s.io/kubernetes/pkg/api/legacyscheme"
 "testing"
 "time"
 )
 
 func newEtcdClient() (*clientv3.Client, error) {
  cfgTLS := &transport.TLSInfo{
   CertFile:      "/etc/kubernetes/pki/etcd/server.crt",
   KeyFile:       "/etc/kubernetes/pki/etcd/server.key",
   TrustedCAFile: "/etc/kubernetes/pki/etcd/ca.crt",
  }
 
  tlsCfg, err := cfgTLS.ClientConfig()
  if err != nil {
   return nil, err
  }
 
  client, err := clientv3.New(clientv3.Config{
   Endpoints:   []string{"192.168.180.130:2379"},
   DialTimeout: 2 * time.Second,
   TLS:         tlsCfg,
  })
  if err != nil {
   return nil, err
  }
 
  return client, nil
 }
 
 type authenticatedDataString string
 
 // AuthenticatedData implements the value.Context interface.
 func (d authenticatedDataString) AuthenticatedData() []byte {
  return []byte(d)
 }
 
 var _ value.Context = authenticatedDataString("")
 
 func TestSecretDecode(t *testing.T) {
  var testSecretKey = "/registry/secrets/rook-ceph/rook-ceph-rgw-token-vd98q"
 
  etcdClient, err := newEtcdClient()
  if err != nil {
   t.Fatal(err)
  }
 
  resp, err := etcdClient.Get(context.TODO(), testSecretKey)
  if err != nil {
   t.Fatal(err)
  }
  encryptedData := resp.Kvs[0].Value
 
  transformers, err := encryptionconfig.GetTransformerOverrides("/etc/kubernetes/pki/secrets-encryption.yaml")
  if err != nil {
   t.Fatal(err)
  }
  gr := schema.ParseGroupResource("secrets")
  transformer := transformers[gr]
  decryptedData, _, err := transformer.TransformFromStorage(encryptedData, authenticatedDataString(testSecretKey))
  if err != nil {
   t.Fatal(err)
  }
 
  var secret v1.Secret
  if _, _, err := legacyscheme.Codecs.UniversalDeserializer().Decode(decryptedData, nil, &secret); err != nil {
   t.Fatal(err)
  }
 
  unst, err := runtime.DefaultUnstructuredConverter.ToUnstructured(&secret)
  if err != nil {
   t.Fatal(err)
  }
  t.Log(unst)
 }
```

**总结**

本文主要是熟悉K8s资源在etcd中的存储方式：

K8s是通过kube-apiserver组件将数据存储在etcd中；

这些数据在写入etcd之前通常会通过protobuf或者json进行序列化；

有的数据在序列化之后是需要加密存储的，这些加密资源是kube-apiserver组件通过--encryption-provider-config参数指定的。