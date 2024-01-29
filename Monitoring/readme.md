# 🚀 Bem-vindo ao Nosso Cluster Kubernetes! 🌟

## Configurando a máquina

1. Instale a ferramenta `eksctl`:

    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    ```

2. Instale o AWS CLI:

    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```

3. Configure as credenciais da AWS:

    ```bash
    aws configure
    ```

## Instalando o EKS

1. Baixe e instale o `eksctl` e o AWS CLI de acordo com o seu sistema operacional.

2. Crie o cluster EKS:

    ```bash
    eksctl create cluster --name=eks-cluster --version=1.24 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t3.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed
    ```

3. Instale o `kubectl`:

    ```bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```

4. Configure o `kubectl` para usar o cluster EKS:

    ```bash
    aws eks --region us-east-1 update-kubeconfig --name eks-cluster
    ```

5. Verifique se o `kubectl` está funcionando corretamente:

    ```bash
    kubectl get nodes
    ```

## Instalando o Kube-Prometheus

1. Clone o repositório e aplique os manifests:

    ```bash
    git clone https://github.com/prometheus-operator/kube-prometheus
    cd kube-prometheus
    kubectl create -f manifests/setup
    ```

2. Verifique a conclusão da instalação dos CRDs:

    ```bash
    kubectl get servicemonitors -A
    ```

3. Instale Prometheus e Alertmanager:

    ```bash
    kubectl apply -f manifests/
    ```

4. Verifique a conclusão da instalação:

    ```bash
    kubectl get pods -n monitoring
    ```

Agora que já temos o nosso Kube-Prometheus instalado, vamos acessar o nosso Grafana e verificar se está tudo funcionando corretamente. Para isso, vamos utilizar o `kubectl port-forward` para acessar o Grafana localmente. Execute o seguinte comando:

```bash
kubectl port-forward -n monitoring svc/grafana 33000:3000
Acesse o Grafana através do navegador: http://localhost:33000
```

Utilize o usuário admin e a senha admin para o primeiro login. O Grafana solicitará a alteração da senha.

Dashboards do Kube-Prometheus
O Kube-Prometheus configura vários Dashboards no Grafana para monitorar o seu cluster EKS.
Alguns exemplos incluem:
Detalhes do API Server e componentes do Kubernetes (Node, Pod, Deployment, etc.).
Detalhes do cluster EKS, como uso de CPU e memória por todos os nós.
Detalhes por namespace, incluindo pods, deployments, statefulsets e daemonsets.
Uso de CPU e memória por nó do cluster EKS.
Uso de CPU e memória por container em todos os pods do cluster EKS.

Brinque com os Dashboards e aproveite a quantidade enorme de informações fornecidas pelo Kube-Prometheus! 🚀


## ServiceMonitors no Kube-Prometheus

O Kube-Prometheus utiliza o recurso `ServiceMonitor` para configurar o Prometheus para monitorar serviços específicos. Já vem configurado com vários `ServiceMonitors`, incluindo os do API Server, Node Exporter, Blackbox Exporter, etc.

Para listar os `ServiceMonitors`:

```bash
kubectl get servicemonitors -n monitoring
```
Para ver detalhes de um ServiceMonitor, como o do Prometheus:

```bash
kubectl get servicemonitor prometheus-k8s -n monitoring -o yaml
```
Um exemplo simples de um ServiceMonitor:

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.41.0
  name: prometheus-k8s
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: web
  - interval: 30s
    port: reloader-web
  selector:
    matchLabels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
```

 Agora pode ser criado os ServiceMonitors para monitorar serviços adicionais do cluster! 🛠️

 # Configuração do ServiceMonitor para Nginx e Nginx Exporter no Kubernetes

## ConfigMap para Nginx
Primeiro, criamos um ConfigMap para a configuração do Nginx. O arquivo YAML é o seguinte:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    # Conteúdo do arquivo de configuração do Nginx
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
      }
      location /metrics {
        stub_status on;
        access_log off;
      }
    }
```

Agora, aplicamos o ConfigMap:
```bash
kubectl apply -f nginx-config.yaml
```

Deployment da Aplicação Nginx

```yaml
apiVersion: apps/v1 # versão da API
kind: Deployment # tipo de recurso, no caso, um Deployment
metadata: # metadados do recurso 
  name: nginx-server # nome do recurso
spec: # especificação do recurso
  selector: # seletor para identificar os pods que serão gerenciados pelo deployment
    matchLabels: # labels que identificam os pods que serão gerenciados pelo deployment
      app: nginx # label que identifica o app que será gerenciado pelo deployment
  replicas: 3 # quantidade de réplicas do deployment
  template: # template do deployment
    metadata: # metadados do template
      labels: # labels do template
        app: nginx # label que identifica o app
      annotations: # annotations do template
        prometheus.io/scrape: 'true' # habilita o scraping do Prometheus
        prometheus.io/port: '9113' # porta do target
    spec: # especificação do template
      containers: # containers do template 
        - name: nginx # nome do container
          image: nginx # imagem do container do Nginx
          ports: # portas do container
            - containerPort: 80 # porta do container
              name: http # nome da porta
          volumeMounts: # volumes que serão montados no container
            - name: nginx-config # nome do volume
              mountPath: /etc/nginx/conf.d/default.conf # caminho de montagem do volume
              subPath: nginx.conf # subpath do volume
        - name: nginx-exporter # nome do container que será o exporter
          image: 'nginx/nginx-prometheus-exporter:0.11.0' # imagem do container do exporter
          args: # argumentos do container
            - '-nginx.scrape-uri=http://localhost/metrics' # argumento para definir a URI de scraping
          resources: # recursos do container
            limits: # limites de recursos
              memory: 128Mi # limite de memória
              cpu: 0.3 # limite de CPU
          ports: # portas do container
            - containerPort: 9113 # porta do container que será exposta
              name: metrics # nome da porta
      volumes: # volumes do template
        - configMap: # configmap do volume, nós iremos criar esse volume através de um configmap
            defaultMode: 420 # modo padrão do volume
            name: nginx-config # nome do configmap
          name: nginx-config # nome do volume
 ```

Agora, aplicamos o Deployment:
```bash
kubectl apply -f nginx-deployment.yaml
```

Service para Expor a Aplicação
```yaml
apiVersion: v1 # versão da API
kind: Service # tipo de recurso, no caso, um Service
metadata: # metadados do recurso
  name: nginx-svc # nome do recurso
  labels: # labels do recurso
    app: nginx # label para identificar o svc
spec: # especificação do recurso
  ports: # definição da porta do svc 
  - port: 9113 # porta do svc
    name: metrics # nome da porta
  selector: # seletor para identificar os pods/deployment que esse svc irá expor
    app: nginx # label que identifica o pod/deployment que será exposto
```
Aplicamos o Service:
```bash
kubectl apply -f nginx-service.yaml
```
Verificação
```bash
curl http://<EXTERNAL-IP-DO-SERVICE>:80
curl http://<EXTERNAL-IP-DO-SERVICE>:80/nginx_status
curl http://<EXTERNAL-IP-DO-SERVICE>:80/metrics
```

Vamos criar o nosso ServiceMonitor com o seguinte arquivo YAML:
```yaml
apiVersion: monitoring.coreos.com/v1 # versão da API
kind: ServiceMonitor # tipo de recurso, no caso, um ServiceMonitor do Prometheus Operator
metadata: # metadados do recurso
  name: nginx-servicemonitor # nome do recurso
  labels: # labels do recurso
    app: nginx # label que identifica o app
spec: # especificação do recurso
  selector: # seletor para identificar os pods que serão monitorados
    matchLabels: # labels que identificam os pods que serão monitorados
      app: nginx # label que identifica o app que será monitorado
  endpoints: # endpoints que serão monitorados
    - interval: 10s # intervalo de tempo entre as requisições
      path: /metrics # caminho para a requisição
      targetPort: 9113 # porta do target
```

Com isso concluimos a configuração do ServiceMonitor para monitorar a aplicação Nginx no Kubernetes.

# Configurando PodMonitors para Métricas não HTTP no Kubernetes

## Introdução
Quando lidamos com workloads que não são serviços HTTP, como CronJobs, Jobs, DaemonSets, entre outros, a monitorização pode ser feita por meio do PodMonitor. Neste guia, veremos como criar um PodMonitor para monitorar um Pod específico, expondo métricas do Nginx e do Nginx Exporter.

## Criando um PodMonitor
Vamos criar o nosso PodMonitor utilizando o arquivo YAML `nginx-pod-monitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1 # versão da API
kind: PodMonitor # tipo de recurso, no caso, um PodMonitor do Prometheus Operator
metadata: # metadados do recurso
  name: nginx-podmonitor # nome do recurso
  labels: # labels do recurso
    app: nginx # label que identifica o app
spec:
  namespaceSelector: # seletor de namespaces
    matchNames: # namespaces que serão monitorados
      - default # namespace que será monitorado
  selector: # seletor para identificar os pods que serão monitorados
    matchLabels: # labels que identificam os pods que serão monitorados
      app: nginx # label que identifica o app que será monitorado
  podMetricsEndpoints: # endpoints que serão monitorados
    - interval: 10s # intervalo de tempo entre as requisições
      path: /metrics # caminho para a requisição
      targetPort: 9113 # porta do target
```
Antes de deployar o nosso PodMonitor, vamos criar o nosso Pod do Nginx com o seguinte arquivo YAML chamado nginx-pod.yaml:

```yaml
apiVersion: v1 # versão da API
kind: Pod # tipo de recurso, no caso, um Pod
metadata: # metadados do recurso
  name: nginx-pod # nome do recurso
  labels: # labels do recurso
    app: nginx # label que identifica o app
spec: # especificações do recursos
  containers: # containers do template 
    - name: nginx-container # nome do container
      image: nginx # imagem do container do Nginx
      ports: # portas do container
        - containerPort: 80 # porta do container
          name: http # nome da porta
      volumeMounts: # volumes que serão montados no container
        - name: nginx-config # nome do volume
          mountPath: /etc/nginx/conf.d/default.conf # caminho de montagem do volume
          subPath: nginx.conf # subpath do volume
    - name: nginx-exporter # nome do container que será o exporter
      image: 'nginx/nginx-prometheus-exporter:0.11.0' # imagem do container do exporter
      args: # argumentos do container
        - '-nginx.scrape-uri=http://localhost/metrics' # argumento para definir a URI de scraping
      resources: # recursos do container
        limits: # limites de recursos
          memory: 128Mi # limite de memória
          cpu: 0.3 # limite de CPU
      ports: # portas do container
        - containerPort: 9113 # porta do container que será exposta
          name: metrics # nome da porta
  volumes: # volumes do template
    - configMap: # configmap do volume, nós iremos criar esse volume através de um configmap
        defaultMode: 420 # modo padrão do volume
        name: nginx-config # nome do configmap
      name: nginx-config # nome do volume
```
Com o Pod criado, agora podemos implantar o PodMonitor:

```bash
kubectl apply -f nginx-pod-monitor.yaml
```
Vamos verificar se o PodMonitor foi criado com sucesso:
```bash
kubectl get podmonitors
```
Se quiser ver detalhes específicos do PodMonitor, execute:
```bash
kubectl describe podmonitor nginx-podmonitor
```
Também é possível verificar informações do ServiceMonitor:
```bash
kubectl describe servicemonitors nginx-servicemonitor
```

Verificando o Funcionamento
Agora, verificamos se o PodMonitor está listado como um target no Prometheus. Para isso, acessamos o Prometheus localmente através do kubectl port-forward:
```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 39090:9090
```

Certifique-se de conferir o seu novo target e as métricas coletadas.

Além disso, é possível acessar o container usando kubectl exec para verificar se o exporter está funcionando corretamente ou para inspecionar as métricas expostas. Execute o seguinte comando:

```bash
kubectl exec -it nginx-pod -c nginx-exporter -- bash
```
Use o curl para verificar se o exporter está funcionando corretamente:

```bash
curl localhost:9113/metrics
```
Caso todas as etapas foram concluídas com êxito, você deve ver uma saída com as métricas do Nginx e do Nginx Exporter.

# Se todas as etapas foram concluídas com êxito, você deve ver uma saída com as métricas do Nginx e do Nginx Exporter.

## Criando nosso primeiro alerta

Agora que já temos o nosso Kube-Prometheus instalado, vamos configurar o Prometheus para monitorar o nosso cluster EKS. Para isso, vamos utilizar o kubectl port-forward para acessar o Prometheus localmente. Para isso, basta executar o seguinte comando:

```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 39090:9090
```
Se você quiser acessar o Alertmanager, basta executar o seguinte comando:
```bash
kubectl port-forward -n monitoring svc/alertmanager-main 39093:9093
```
Pronto, agora você já sabe como que faz para acessar o Prometheus, AlertManager e o Grafana localmente. 😄

Lembrando que você pode acessar o Prometheus e o AlertManager através do seu navegador, basta acessar as seguintes URLs:

Prometheus: http://localhost:39090
AlertManager: http://localhost:39093

Simples assim!

## Entendendo PrometheusRule
Antes de sair definindo um novo alerta, precisamos entender como fazê-lo, uma vez que nós não temos mais o arquivo de alertas, igual tínhamos quando instalamos o Prometheus em nosso servidor Linux.

Agora, precisamos entender que boa parte da configuração do Prometheus está dentro de configmaps, que são recursos do Kubernetes que armazenam dados em formato de chave e valor e são muito utilizados para armazenar configurações de aplicações.

Para listar os configmaps do nosso cluster, basta executar o seguinte comando:
```bash
kubectl get configmaps -n monitoring
```
O resultado do comando acima deverá ser parecido com o seguinte:

```bash
NAME                                                  DATA   AGE
adapter-config                                        1      7m20s
blackbox-exporter-configuration                       1      7m49s
# ... (outros configmaps)
prometheus-k8s-rulefiles-0                            8      7m10s
```
Como você pode ver, temos diversos configmaps que contêm configurações do Prometheus, AlertManager e do Grafana. Vamos focar no configmap prometheus-k8s-rulefiles-0, que é o configmap que contém os alertas do Prometheus.

Para visualizar o conteúdo do configmap, basta executar o seguinte comando:
```bash
kubectl get configmap prometheus-k8s-rulefiles-0 -n monitoring -o yaml
```
## Criando um novo alerta
Como exemplo, já temos um alerta chamado KubeMemoryOvercommit. Vamos criar um novo alerta para monitorar o estado do Nginx.

## O que é um PrometheusRule?
O PrometheusRule é um recurso do Kubernetes que foi instalado no momento que realizamos a instalação dos CRDs do kube-prometheus. O PrometheusRule permite que você defina alertas para o Prometheus. Ele é muito parecido com o arquivo de alertas que criamos no nosso servidor Linux, porém, nesse momento, vamos fazer a mesma definição de alerta, mas usando o PrometheusRule.

# Criando um PrometheusRule

Vamos criar um arquivo chamado nginx-prometheus-rule.yaml e vamos colocar o seguinte conteúdo:

```yaml
apiVersion: monitoring.coreos.com/v1 # Versão da api do PrometheusRule
kind: PrometheusRule # Tipo do recurso
metadata: # Metadados do recurso (nome, namespace, labels)
  name: nginx-prometheus-rule
  namespace: monitoring
  labels: # Labels do recurso
    prometheus: k8s # Label que indica que o PrometheusRule será utilizado pelo Prometheus do Kubernetes
    role: alert-rules # Label que indica que o PrometheusRule contém regras de alerta
    app.kubernetes.io/name: kube-prometheus # Label que indica que o PrometheusRule faz parte do kube-prometheus
    app.kubernetes.io/part-of: kube-prometheus # Label que indica que o PrometheusRule faz parte do kube-prometheus
spec: # Especificação do recurso
  groups: # Lista de grupos de regras
  - name: nginx-prometheus-rule # Nome do grupo de regras
    rules: # Lista de regras
    - alert: NginxDown # Nome do alerta
      expr: up{job="nginx"} == 0 # Expressão que será utilizada para disparar o alerta
      for: 1m # Tempo que a expressão deve ser verdadeira para que o alerta seja disparado
      labels: # Labels do alerta
        severity: critical # Label que indica a severidade do alerta
      annotations: # Anotações do alerta
        summary: "Nginx is down" # Título do alerta
        description: "Nginx is down for more than 1 minute. Pod name: {{ $labels.pod }}" # Descrição do alerta

```

Agora, vamos criar o PrometheusRule no nosso cluster:

```bash
kubectl apply -f nginx-prometheus-rule.yaml
```

Agora, vamos verificar se o PrometheusRule foi criado com sucesso:

```bash
kubectl get prometheusrules -n monitoring
```
A saída deve ser parecida com essa:

```bash
NAME                              AGE
alertmanager-main-rules           2m
grafana-rules                     2m
kube-prometheus-rules             2m
kube-state-metrics-rules          2m
kubernetes-monitoring-rules       2m
nginx-prometheus-rule             2s
node-exporter-rules               1m
prometheus-k8s-prometheus-rules   9m
prometheus-operator-rules         1m

```
Agora nós já temos um novo alerta configurado em nosso Prometheus. Lembrando que temos a integração com o AlertManager, então, quando o alerta for disparado, ele será enviado para o AlertManager, e o AlertManager vai enviar uma notificação, por exemplo, para o nosso Slack ou e-mail.

Você pode acessar o nosso alerta tanto no Prometheus quanto no AlertManager.

Vamos imaginar que você precisa criar um novo alerta para monitorar a quantidade de requisições simultâneas que o seu Nginx está recebendo. Para isso, você precisa criar uma nova regra no PrometheusRule. Podemos utilizar o mesmo arquivo nginx-prometheus-rule.yaml e adicionar a nova regra no final do arquivo:

```yaml
apiVersion: monitoring.coreos.com/v1 # Versão da api do PrometheusRule
kind: PrometheusRule # Tipo do recurso
metadata: # Metadados do recurso (nome, namespace, labels)
  name: nginx-prometheus-rule
  namespace: monitoring
  labels: # Labels do recurso
    prometheus: k8s # Label que indica que o PrometheusRule será utilizado pelo Prometheus do Kubernetes
    role: alert-rules # Label que indica que o PrometheusRule contém regras de alerta
    app.kubernetes.io/name: kube-prometheus # Label que indica que o PrometheusRule faz parte do kube-prometheus
    app.kubernetes.io/part-of: kube-prometheus # Label que indica que o PrometheusRule faz parte do kube-prometheus
spec: # Especificação do recurso
  groups: # Lista de grupos de regras
  - name: nginx-prometheus-rule # Nome do grupo de regras
    rules: # Lista de regras
    - alert: NginxDown # Nome do alerta
      expr: up{job="nginx"} == 0 # Expressão que será utilizada para disparar o alerta
      for: 1m # Tempo que a expressão deve ser verdadeira para que o alerta seja disparado
      labels: # Labels do alerta
        severity: critical # Label que indica a severidade do alerta
      annotations: # Anotações do alerta
        summary: "Nginx is down" # Título do alerta
        description: "Nginx is down for more than 1 minute. Pod name: {{ $labels.pod }}" # Descrição do alerta

    - alert: NginxHighRequestRate # Nome do alerta
        expr: rate(nginx_http_requests_total{job="nginx"}[5m]) > 10 # Expressão que será utilizada para disparar o alerta
        for: 1m # Tempo que a expressão deve ser verdadeira para que o alerta seja disparado
        labels: # Labels do alerta
            severity: warning # Label que indica a severidade do alerta
        annotations: # Anotações do alerta
            summary: "Nginx is receiving high request rate" # Título do alerta
            description: "Nginx is receiving high request rate for more than 1 minute. Pod name: {{ $labels.pod }}" # Descrição do alerta
```
foi criado uma nova definição de alerta em nosso PrometheusRule. Agora vamos atualizar o nosso PrometheusRule

```bash
kubectl apply -f nginx-prometheus-rule.yaml
```
podemos verificar se o PrometheusRule foi atualizado:
```bash
kubectl get prometheusrules -n monitoring nginx-prometheus-rule -o yaml
```
Com a integração do AlertManager, ao receber mais de 10 requisições por minuto, o alerta será acionado, enviando uma notificação para o Slack ou e-mail, dependendo da configuração no AlertManager.