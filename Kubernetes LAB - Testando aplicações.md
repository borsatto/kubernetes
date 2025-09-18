[[Kubernetes LAB - Ubuntu 24.04]]
### Tarefa 1: Escalar o Deployment e Entender o Balanceamento de Carga

**Objetivo:** Escalar o Deployment `nginx-lb` para 4 réplicas e observar como o Kubernetes gerencia o balanceamento de carga entre os pods.

**Solução:**

1. **Escalar o Deployment:**
   Execute o seguinte comando para escalar o Deployment `nginx-lb` para 4 réplicas:
   ```bash
   kubectl scale deployment nginx-lb --replicas=4
   ```

2. **Verificar o Status dos Pods:**
   Após escalar, verifique o status dos pods:
   ```bash
   kubectl get pods -o wide
   ```
   Você verá que agora há 4 pods em execução, distribuídos entre os nós workers.

3. **Acessar o Serviço:**
   Acesse o serviço `nginx-lb-service` usando o IP externo (`192.168.178.80`) no navegador ou via `curl`:
   ```bash
   curl http://192.168.178.80
   ```
   O MetalLB e o NGINX já estão configurados para balancear a carga entre os pods.

**Explicação:**
- **Escalabilidade:** O Kubernetes permite escalar facilmente o número de réplicas de um Deployment. Isso é útil para lidar com aumentos de tráfego ou para garantir alta disponibilidade.
- **Balanceamento de Carga:** O Kubernetes, em conjunto com o MetalLB e o NGINX, distribui as requisições entre os pods. Cada requisição pode ser atendida por um pod diferente, garantindo que a carga seja distribuída uniformemente.

### Tarefa 2: Configurar um ConfigMap para Personalizar o NGINX

**Objetivo:** Criar um ConfigMap para personalizar a página inicial do NGINX e montá-lo no Deployment.

**Solução:**

1. **Criar um ConfigMap:**
   Crie um arquivo `index.html` com o conteúdo personalizado:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <title>Meu NGINX Personalizado</title>
   </head>
   <body>
       <h1>Bem-vindo ao Meu NGINX Personalizado!</h1>
   </body>
   </html>
   ```
   Agora, crie um ConfigMap a partir desse arquivo:
   ```bash
   kubectl create configmap nginx-index --from-file=index.html
   ```

[[Kubernetes LAB - Como adicionar linhas no ConfigMap]]

2. **Atualizar o Deployment:**
   Edite o Deployment `nginx-lb` para montar o ConfigMap como um volume:
   ```yaml
   kubectl edit deployment nginx-lb
   ```
   Adicione as seguintes linhas na seção `spec.template.spec.containers.volumeMounts` e `spec.template.spec.volumes`:
   ```yaml
   volumeMounts:
   - name: nginx-index
     mountPath: /usr/share/nginx/html
   volumes:
   - name: nginx-index
     configMap:
       name: nginx-index
   ```

3. **Verificar a Atualização:**
   Após salvar as alterações, o Kubernetes recriará os pods com o novo ConfigMap montado. Acesse novamente o serviço para ver a página personalizada.

**Explicação:**
- **ConfigMap:** O ConfigMap é usado para armazenar dados de configuração não confidenciais, como arquivos de configuração ou conteúdo estático. Neste caso, usamos para personalizar a página inicial do NGINX.
- **Volumes:** O Kubernetes permite montar ConfigMaps como volumes nos pods, o que é útil para injetar configurações ou conteúdo estático nos contêineres.

### Tarefa 3: Implementar um HPA (Horizontal Pod Autoscaler)

**Objetivo:** Configurar um Horizontal Pod Autoscaler (HPA) para escalar automaticamente o Deployment `nginx-lb` com base no uso de CPU.

**Solução:**

1. **Habilitar o Metrics Server:**
   Certifique-se de que o Metrics Server está instalado no cluster. Se não estiver, você pode instalá-lo com:
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

2. **Criar o HPA:**
   Crie um HPA para o Deployment `nginx-lb`:
   ```bash
   kubectl autoscale deployment nginx-lb --cpu-percent=50 --min=2 --max=6
   ```
   Isso configurará o HPA para escalar o Deployment entre 2 e 6 réplicas, mantendo o uso médio da CPU abaixo de 50%.

3. **Verificar o HPA:**
   Verifique o status do HPA:
   ```bash
   kubectl get hpa
   ```
   Você verá o uso atual da CPU e o número desejado de réplicas.

4. **Gerar Carga para Testar o HPA:**
   Para testar o HPA, você pode gerar carga no serviço usando uma ferramenta como `ab` (Apache Benchmark):
   ```bash
   ab -n 100000 -c 100 http://192.168.178.80/
   ```
   Observe como o HPA ajusta o número de réplicas com base no uso da CPU.
[[Kubernetes LAB - Teste de carga em servidor Web]]

**Explicação:**
- **HPA:** O Horizontal Pod Autoscaler ajusta automaticamente o número de réplicas de um Deployment com base na utilização de recursos, como CPU ou memória. Isso ajuda a garantir que o aplicativo tenha recursos suficientes para lidar com a carga, sem sobrecarregar os nós.
- **Metrics Server:** O Metrics Server coleta métricas de uso de recursos dos pods e nós, permitindo que o HPA tome decisões de escalonamento com base em dados reais.
