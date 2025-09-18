
Para essa aplicação de teste, podemos usar uma stack simples, como:

- **Frontend:** React com um formulário para editar os dados.
- **Backend:** Node.js com Express para receber e salvar os dados.
- **Banco de Dados:** PostgreSQL ou MySQL para armazenar os dados.

Vamos instalar o **MySQL** no seu cluster Kubernetes. Podemos fazer criando os manifests YAML manualmente (mais controle).

Nesta abordagem, vamos criar os manifests **Deployment + Service + PersistentVolumeClaim + PersistentVolume** para rodar o MySQL.

### 1️⃣Criar o PersistentVolume (PV)

Crie um novo arquivo chamado `mysql-pv.yaml` e adicione:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # Evita que o PV seja apagado ao deletar o PVC
  storageClassName: manual
  hostPath:
    path: "/mnt/mysql-data"  # Caminho no host onde os dados serão salvos
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

🔹 **`hostPath`** define o local no host onde os dados ficarão armazenados permanentemente.  
🔹 **`persistentVolumeReclaimPolicy: Retain`** garante que os dados não sejam apagados mesmo que o PVC seja deletado.

Aplique esse arquivo:
```bash
kubectl apply -f mysql-pv.yaml
```

### 2️⃣ Criar um arquivo YAML para o MySQL

Crie um arquivo chamado `mysql-deployment.yaml` com o seguinte conteúdo:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:8.0
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpassword"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```


### **3️⃣ Aplicar os arquivos no Kubernetes**

Agora, aplique os manifests:
```shell
kubectl apply -f mysql-deployment.yaml
```

### **4️⃣ Verificar se o MySQL está rodando**

Confira os pods e serviços:
```shell
kubectl get pods kubectl get svc
```

O serviço **mysql** estará rodando dentro do cluster.