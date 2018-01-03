## Day 16 - ConfigMap

### 本日共賞

* ConfigMap

### 希望你知道
* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### ConfigMap

從字面上來看不難理解 Config 即描述系統相關設定的內容，Map 則是使用 key-vaule 的方式做描述。在 k8s 中，應用程式真正運作在 Pod 內，與應用程式相關的設定則可放在 ConfigMap 內。使用 ConfigMap 儲存設定的方式是希望讓與應用程式與設定解耦 (decoupled)。意思是，ConfigMap 與 Pod 可以單獨存在於 k8s 叢集中，當 Pod 需要使用 ConfigMap 時才需要將 ConfigMap 掛載到 Pod 內使用。

> 解耦的設計會帶來不少好處
> 
> 1. `便於管理`
> 2. `高彈性`：易於掛載不同的 ConfigMap 到 Pod 內使用
> 3. `一處編輯多處使用`：同一個 ConfigMap 可掛載到多個 Pod 使用

在過去使用單體式應用程式 (Monolithic) 時，我們通常都會直接把程式相關的設定寫死 (hardcoded) 在應用程式內，在應用程式不大的時候或許不會有什麼問題產生。但隨著應用程式慢慢長大，同樣的設定可能需要一直重複被使用或修改，這時候維護設定就會變成很麻煩的問題。更不用說在微服務 (Microservices) 的架構下，動輒幾十幾百個微服務在系統內運作，同一份系統設定是很常見的。也因此，ConfigMap 在大型系統的架構下，顯得特別的有用。

> 想想在微服務架構下，當你有超過 20 微服務需要維護時，如果所有的設定都要一個一個改，維護的工作有多吃重。

k8s 中的 ConfigMap 可以接受兩種來源，一個是 `--from-literal`，一個是`--from-file`。

**--from-literal**

讓我們先來看看 `--from-literal` 的方式如何建立 ConfigMap

```bash
$ kubectl create configmap myconfig --from-literal=k1=v1 --from-literal=k2=v2
configmap "myconfig" created
```

* `kubectl create configmap [name]`：`name` 即為 ConfigMap 的名稱，以上面例子來看就是 `myconfig`
* `--from-literal=[key]=[value]`：透過 `--from-literal` 參數用文字的方式指定 `[key]` 與 `[value]`，上面例子指定了兩個內容 `k1:v1` 與 `k2:v2`。

接下來查看一下 `myconfig` 的內容，我們將輸出格式設成 `yaml` 方便觀看

```bash
$ kubectl get configmaps myconfig -o yaml  <=== -o 指定輸出格式為 yaml
apiVersion: v1
data:
  k1: v1   <=== 這是我們指定的 key:value
  k2: v2   <=== 這是我們指定的第二個 key:value
kind: ConfigMap
metadata:
  creationTimestamp: 2018-01-03T02:48:29Z
  name: myconfig
  namespace: default  <=== 沒有指定命名空間，所以使用預設的 default
  resourceVersion: "113617"
  selfLink: /api/v1/namespaces/default/configmaps/myconfig
  uid: 943af22d-f030-11e7-bdb7-080027004f24
```

> 還記得 [Day 6 - 與 k8s 溝通: APIs](https://ithelp.ithome.com.tw/articles/10193489) 提到過的每個物件都可以透過 api 存取嗎？你可以試試 [Day 6 - 與 k8s 溝通: APIs](https://ithelp.ithome.com.tw/articles/10193489) 提到的方法存取 `selfLink` 的內容

**--from-file**

第二種方法 `--from-file`，先把 key:value 定義在檔案內，底下是名為 `from-key` 的檔案內容，我們一共建立三組 key:value

```
fromkey1=v1
fromkey2=v2
fromkey3=v3
```

建立 ConfigMap 的方式為

```bash
$ kubectl create configmap myconfigfromkey --from-file=fromfilekey=from-key
```

`--from-key=[key]=[file path]`：即指定 key 綁定的檔案位置，上面指令會產生一個 key 為 `fromfilekey` 並綁定 `from-key` 內容且名為 `myconfigfromkey ` 的 ConfigMap。

觀察一下內容：

```bash
$ kubectl get configmaps myconfigfromkey -o yaml
apiVersion: v1
data:
  fromfilekey: |   <=== | 表示保留换行符，即多行內容
    fromkey1=v1
    fromkey2=v2
    fromkey3=v3
kind: ConfigMap
metadata:
  creationTimestamp: 2018-01-03T02:50:41Z
  name: myconfigfromkey
  namespace: default
  resourceVersion: "113780"
  selfLink: /api/v1/namespaces/default/configmaps/myconfigfromkey
  uid: e2c22b2c-f030-11e7-bdb7-080027004f24
```

另外你也可以直接撰寫 yaml 檔如下：

```
# configmap.yaml

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigyaml
data:
  k1: v1   <=== 這裡可以填寫需要用到的 key:value
  k2: v2
```

接著部署到 k8s 中

```bash
$ kubectl apply -f configmap.yaml
configmap "myconfigyaml" configured
```

> 有關 ConfigMap 的說明可以參考 [官方文件](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/#understanding-configmaps)

有了 ConfigMap 之後該如何使用呢？就像上面提到的我們可以把它掛到 Pod 內使用。底下是一個將 ConfigMap 內的 key 直掛在 Pod 環境變數的例子

```
# configmap.pod.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: config-pod
    image: gcr.io/google_containers/busybox
    command: ["/bin/sh", "-c", "env"]   <=== 執行指令輸出環境變數
    env:
    - name: MY_CONFIG_KEY   <=== 指定一個新的環境變數名稱為 MY_CONFIG_KEY
      valueFrom:
        configMapKeyRef:    <=== MY_CONFIG_KEY 將會參考 myconfig 內的 k1 值
          name: myconfig    <=== myconfig 是我們上面宣告的 ConfigMap 物件，還記得嗎？
          key: k1
  restartPolicy: Never   <=== 只需要執行一次所以不需要重啟
```

我們把它部署到 k8s 看看會發生什麼事情

```bash
$ kubectl apply -f configmap.pod.yam
pod "config-pod" created

$ kubectl get pods   <=== 查看一下發現還在建立容器
NAME                          READY     STATUS              RESTARTS   AGE
config-pod                    0/1       ContainerCreating   0          2s

$ kubectl get pods   <=== 再查看一下發現消失不見了！
NAME                          READY     STATUS              RESTARTS   AGE
```

你會發現 config-pod 會一下就消失不見，那是因為我們只需要它印出環境變數 `command: ["/bin/sh", "-c", "env"]`後就結束，也不需要再重新啟動 `restartPolicy: Never`。

接著可以透過 `kubectl logs` 查看

```bash
$ kubectl logs config-pod
...
MY_CONFIG_KEY=v1
```

你應該可以看到 `MY_CONFIG_KEY=v1`，而 `MY_CONFIG_KEY` 是我們在 yaml 檔內宣告 Pod 時所指定的 key 而 `v1` 則是上面設定 ConfigMap 的時候宣告的內容。k8s 就是透過這樣的方式，讓 Pod 與 ConfigMap 解耦。是不是覺得很有趣啊！

> 你可以試著改變一下 `configMapKeyRef.name` 或 `configMapKeyRef.key` 參考的物件看看是否有發生變化

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day16/](https://jlptf.github.io/ironman2018-day16/)
