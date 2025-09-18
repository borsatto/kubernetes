Se você já tem um arquivo de configuração (`metallb-config.yaml`) com o range de IPs correto, basta aplicar esse arquivo novamente usando o comando:

```bash
kubectl apply -f metallb-config.yaml
```

O Kubernetes é declarativo, então ao executar esse comando, ele atualiza a configuração do MetalLB com base no que está definido no arquivo YAML. O MetalLB, por sua vez, reconhece as mudanças e começa a usar o novo range de IPs imediatamente.

---

### Resumo do processo:
1. **Edite o arquivo `metallb-config.yaml`** para corrigir o range de IPs.
2. **Aplique a configuração** com o comando `kubectl apply -f metallb-config.yaml`.
3. **Verifique os serviços** para confirmar que os IPs estão sendo atribuídos corretamente:

   ```bash
   kubectl get svc
   ```

4. **Reinicie os pods do MetalLB** (se necessário) para garantir que as mudanças sejam aplicadas imediatamente:

   ```bash
   kubectl delete pod -n metallb-system --all
   ```

5. **Verifique os logs** (se algo não estiver funcionando):

   ```bash
   kubectl logs -n metallb-system -l app=metallb
   ```

---

### Por que isso funciona?
O MetalLB usa um `ConfigMap` para armazenar sua configuração. Quando você aplica um arquivo YAML com `kubectl apply`, o Kubernetes atualiza o `ConfigMap` e notifica o MetalLB sobre a mudança. O MetalLB, então, recarrega a configuração e começa a usar o novo range de IPs.

