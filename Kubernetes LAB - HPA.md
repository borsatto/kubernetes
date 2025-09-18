Para alterar o **Horizontal Pod Autoscaler (HPA)** de um serviço, como reduzir o `--cpu-percent` de 50% para 40%, você pode usar o comando `kubectl edit` ou `kubectl patch`. Vou explicar ambos os métodos.

---

### Método 1: Usando `kubectl edit`

1. **Editar o HPA diretamente:**
   Execute o seguinte comando para editar o HPA:
   ```bash
   kubectl edit hpa <nome-do-hpa>
   ```
   Substitua `<nome-do-hpa>` pelo nome do HPA que você deseja modificar.

2. **Localizar a configuração de CPU:**
   No editor que abrir, localize a linha que contém `targetCPUUtilizationPercentage` ou `metrics` (dependendo da versão do Kubernetes). Altere o valor de `50` para `40`.

   Exemplo:
   ```yaml
   spec:
     metrics:
     - resource:
         name: cpu
         target:
           averageUtilization: 40
           type: Utilization
   ```

3. **Salvar e sair:**
   Salve as alterações e saia do editor. O Kubernetes atualizará automaticamente o HPA com as novas configurações.

---

### Método 2: Usando `kubectl patch`

Se você preferir não abrir um editor, pode usar o comando `kubectl patch` para alterar o HPA diretamente.

1. **Patch no HPA:**
   Execute o seguinte comando para alterar o `targetCPUUtilizationPercentage` para 40%:
   ```bash
   kubectl patch hpa <nome-do-hpa> -p '{"spec":{"targetCPUUtilizationPercentage":40}}'
   ```
   Substitua `<nome-do-hpa>` pelo nome do HPA.

   **Nota:** Se você estiver usando uma versão mais recente do Kubernetes (com a API `autoscaling/v2`), o campo `targetCPUUtilizationPercentage` foi substituído por `metrics`. Nesse caso, use o seguinte comando:
   ```bash
   kubectl patch hpa <nome-do-hpa> -p '{"spec":{"metrics":[{"resource":{"name":"cpu","target":{"averageUtilization":40,"type":"Utilization"}}]}}'
   ```

2. **Verificar a alteração:**
   Após aplicar o patch, verifique se o HPA foi atualizado corretamente:
   ```bash
   kubectl get hpa <nome-do-hpa> -o yaml
   ```

---

### Método 3: Recriar o HPA

Se preferir recriar o HPA do zero, você pode excluir o HPA existente e criar um novo com os parâmetros desejados.

1. **Excluir o HPA existente:**
   ```bash
   kubectl delete hpa <nome-do-hpa>
   ```

2. **Criar um novo HPA:**
   Use o comando `kubectl autoscale` para criar um novo HPA com o `--cpu-percent` ajustado para 40%:
   ```bash
   kubectl autoscale deployment <nome-do-deployment> --cpu-percent=40 --min=<min-replicas> --max=<max-replicas>
   ```
   Substitua `<nome-do-deployment>`, `<min-replicas>` e `<max-replicas>` pelos valores apropriados.

   Exemplo:
   ```bash
   kubectl autoscale deployment nginx-lb --cpu-percent=40 --min=2 --max=6
   ```

---

### Verificando o HPA

Após fazer as alterações, verifique o status do HPA para garantir que ele foi atualizado corretamente:

```bash
kubectl get hpa
```

A saída mostrará o novo valor de `TARGETS` (por exemplo, `40%/40%`).

---

### Explicação dos Campos

- **`targetCPUUtilizationPercentage` (ou `averageUtilization`):**
  - Define o percentual de utilização da CPU que o HPA deve manter. Se a utilização da CPU exceder esse valor, o HPA escalará o número de réplicas.

- **`min` e `max`:**
  - Define o número mínimo e máximo de réplicas que o HPA pode escalar.

---

### Dica: Usando `kubectl describe`

Se quiser mais detalhes sobre o HPA, use o comando `kubectl describe`:

```bash
kubectl describe hpa <nome-do-hpa>
```

Isso mostrará informações detalhadas, como eventos, métricas atuais e ações de escalonamento.

---

O erro que você está enfrentando indica que o **Horizontal Pod Autoscaler (HPA)** não consegue acessar as métricas de CPU, o que impede o autoscaling. Esse problema geralmente ocorre porque o **Metrics Server** não está instalado ou não está funcionando corretamente no seu cluster Kubernetes.



---

### Passo 1: Verificar se o Metrics Server Está Instalado

O Metrics Server é responsável por coletar métricas de uso de recursos (CPU e memória) dos pods e nós, e fornecer essas métricas para o HPA.

1. Verifique se o Metrics Server está instalado:
   ```bash
   kubectl get deployment metrics-server -n kube-system
   ```

   Se não houver resultados, o Metrics Server não está instalado.

2. Verifique se há algum pod do Metrics Server em execução:
   ```bash
   kubectl get pods -n kube-system | grep metrics-server
   ```

   Se não houver pods em execução, o Metrics Server não está funcionando.

---

### Passo 2: Instalar o Metrics Server

Se o Metrics Server não estiver instalado, siga estas etapas para instalá-lo:

1. Baixe o arquivo de instalação do Metrics Server:
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

2. Verifique se o Metrics Server foi instalado corretamente:
   ```bash
   kubectl get deployment metrics-server -n kube-system
   ```

3. Verifique os logs do Metrics Server para garantir que ele está funcionando:
   ```bash
   kubectl logs -n kube-system -l k8s-app=metrics-server
   ```

---

### Passo 3: Configurar o Metrics Server (Se Necessário)

Em alguns casos, o Metrics Server pode precisar de configurações adicionais para funcionar corretamente. Por exemplo, se o seu cluster usa certificados autoassinados, você pode precisar adicionar a flag `--kubelet-insecure-tls` ao Metrics Server.

1. Edite o deployment do Metrics Server:
   ```bash
   kubectl edit deployment metrics-server -n kube-system
   ```

2. Adicione a flag `--kubelet-insecure-tls` na seção `args` do contêiner:
   ```yaml
   spec:
     containers:
     - args:
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
       - --cert-dir=/tmp
       - --secure-port=4443
   ```

3. Salve e saia do editor. O Metrics Server será reiniciado com as novas configurações.

4. Verifique se o Metrics Server está funcionando:
   ```bash
   kubectl get pods -n kube-system | grep metrics-server
   ```

---

### Passo 4: Verificar as Métricas

Após instalar e configurar o Metrics Server, verifique se ele está coletando métricas corretamente:

1. Verifique as métricas dos nós:
   ```bash
   kubectl top nodes
   ```

2. Verifique as métricas dos pods:
   ```bash
   kubectl top pods
   ```

   Se esses comandos retornarem métricas, o Metrics Server está funcionando corretamente.

---

### Passo 5: Verificar o HPA

Agora que o Metrics Server está funcionando, verifique o status do HPA:

1. Verifique o status do HPA:
   ```bash
   kubectl get hpa
   ```

2. Verifique os detalhes do HPA:
   ```bash
   kubectl describe hpa <nome-do-hpa>
   ```

   O campo `TARGETS` deve mostrar a utilização atual da CPU (por exemplo, `50%/40%`).

---

### Passo 6: Testar o Autoscaling

Para garantir que o HPA está funcionando corretamente, você pode gerar carga no seu serviço e observar o comportamento do HPA.

1. Gere carga usando uma ferramenta como `ab` (Apache Benchmark):
   ```bash
   ab -n 100000 -c 100 http://<endereço-do-servico>/
   ```

2. Verifique o número de réplicas do deployment:
   ```bash
   kubectl get deployment <nome-do-deployment>
   ```

   O número de réplicas deve aumentar conforme a utilização da CPU.

3. Verifique os eventos do HPA:
   ```bash
   kubectl describe hpa <nome-do-hpa>
   ```

   Você verá eventos como `ScalingActive` e `SuccessfulRescale`.

---

### Resumo das Causas Comuns do Problema

1. **Metrics Server não instalado:**
   - O Metrics Server é necessário para fornecer métricas de CPU e memória ao HPA.

2. **Problemas de configuração do Metrics Server:**
   - Certificados autoassinados ou problemas de rede podem impedir o Metrics Server de coletar métricas.

3. **Problemas de permissão:**
   - Verifique se o Metrics Server tem permissão para acessar a API do Kubernetes.

4. **Problemas de rede:**
   - Verifique se há problemas de conectividade entre o Metrics Server e os nós do cluster.

---

### Resumo do Comportamento do HPA

1. **Escalonamento para Cima (Scale-Up):**
   - Quando a utilização da CPU (ou outra métrica configurada) excede o valor definido no `targetCPUUtilizationPercentage`, o HPA aumenta o número de réplicas para lidar com a carga.

2. **Escalonamento para Baixo (Scale-Down):**
   - Quando a utilização da CPU cai abaixo do valor definido, o HPA reduz o número de réplicas gradualmente, após o tempo de resfriamento (default: 5 minutos).

3. **Tempo de Resfriamento (Cool Down):**
   - O tempo de resfriamento evita flutuações frequentes no número de réplicas, garantindo estabilidade no sistema.

---

### Próximos Passos para Aprofundar seus Conhecimentos

Agora que você já viu o HPA em ação, aqui estão algumas sugestões para continuar explorando e aprofundando seus conhecimentos:

#### 1. **Ajustar o Comportamento do Scale-Down**
   - Experimente reduzir o tempo de resfriamento ou configurar políticas personalizadas de scale-down usando a seção `behavior` no HPA.
   - Exemplo:
     ```yaml
     behavior:
       scaleDown:
         stabilizationWindowSeconds: 60  # Reduz o tempo de resfriamento para 1 minuto
         policies:
         - type: Percent
           value: 50
           periodSeconds: 60
     ```

#### 2. **Usar Outras Métricas para Escalonamento**
   - Além da CPU, você pode configurar o HPA para usar métricas de memória, métricas personalizadas ou até mesmo métricas externas.
   - Exemplo de configuração com métricas de memória:
     ```yaml
     metrics:
     - type: Resource
       resource:
         name: memory
         target:
           type: Utilization
           averageUtilization: 50
     ```

#### 3. **Testar com Outras Aplicações**
   - Experimente configurar o HPA para outras aplicações, como bancos de dados (MySQL, PostgreSQL), filas de mensagens (RabbitMQ, Kafka) ou microsserviços.

#### 4. **Monitorar o HPA com Prometheus e Grafana**
   - Configure o Prometheus e o Grafana para monitorar as métricas de CPU, memória e o número de réplicas em tempo real.
   - Use dashboards personalizados para visualizar o comportamento do HPA.

#### 5. **Simular Cenários de Falha**
   - Teste como o HPA se comporta em cenários de falha, como:
     - Um nó do cluster ficar indisponível.
     - O Metrics Server parar de funcionar.
     - A aplicação não conseguir escalar devido a limites de recursos.

---

### Dicas para Ambientes de Produção

1. **Defina Limites de Recursos (Requests e Limits):**
   - Sempre defina `requests` e `limits` para CPU e memória nos seus pods. Isso ajuda o HPA a tomar decisões mais precisas.

2. **Use Namespaces e Quotas:**
   - Organize seus recursos usando namespaces e defina quotas de recursos para evitar que uma aplicação consuma todos os recursos do cluster.

3. **Monitore e Ajuste:**
   - Monitore o comportamento do HPA regularmente e ajuste os parâmetros conforme necessário para otimizar o desempenho e o custo.
