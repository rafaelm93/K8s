# Ingress no K8s

### Usando KinD com Ingress

## Introdução

O KinD (Kubernetes in Docker) é uma ferramenta valiosa para testes e desenvolvimento com Kubernetes. Nesta seção atualizada, fornecemos detalhes específicos para garantir que o Ingress funcione como esperado em um cluster KinD.

## Criando o Cluster com Configurações Especiais

Ao criar um cluster KinD, podemos especificar várias configurações, incluindo mapeamentos de portas e rótulos para nós.

1. Crie um arquivo chamado `kind-config.yaml` com o seguinte conteúdo:

    ```yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
      extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
    ```

2. Em seguida, crie o cluster usando este arquivo de configuração:

    ```bash
    kind create cluster --config kind-config.yaml
    ```

## Instalando um Ingress Controller

Vamos continuar usando o Nginx Ingress Controller como exemplo, que é amplamente adotado e bem documentado.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```
 pode ser utilizado a opção wait do kubectl, assim quando os pods estiverem prontos, ele liberará o shell, veja:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

```

No comando acima, estamos aguardando que os pods do Ingress Controller estejam prontos, com o label app.kubernetes.io/component=controller, no namespace ingress-nginx. Caso não estejam prontos em 90 segundos, o comando irá falhar.

# Configurando Ingress Controller em Diferentes Ambientes Kubernetes

## Minikube

Para instalar o Ingress Controller no Minikube, você pode usar o sistema de addons do próprio Minikube. Execute o seguinte comando:

```bash
minikube addons enable ingress
```
## MicroK8s
No MicroK8s, o Ingress Controller também pode ser instalado via sistema de addons. Use o seguinte comando:
```bash
microk8s enable ingress
```
## AWS (Amazon Web Services)
Na AWS, o Ingress Controller é exposto por meio de um Network Load Balancer (NLB). O comando para instalação é:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/aws/deploy.yaml
```
também pode ser configurado a terminação TLS no Load Balancer da AWS, editando o arquivo deploy.yaml com as informações do seu Certificate Manager (ACM).

## Azure
No Azure, o Ingress Controller é instalado com o comando:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
## GCP (Google Cloud Platform)
No GCP, primeiro certifique-se de que seu usuário tem permissões de cluster-admin. Depois, instale o Ingress Controller com:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
## Bare Metal
Para instalações em Bare Metal ou VMs "cruas", você pode usar um NodePort para testes rápidos:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```

## app-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        env:
        - name: REDIS_HOST
          value: redis-service
        ports:
        - containerPort: 5000
        imagePullPolicy: Always
```
## app-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
      name: tcp-app
  type: ClusterIP
```
## redis-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - image: redis
        name: redis
        ports:
          - containerPort: 6379
        resources:
          limits:
            memory: "256Mi"
            cpu: "500m"
          requests:
            memory: "128Mi"
            cpu: "250m"
```
## redis-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
  type: ClusterIP

```

## Criando um Recurso de Ingress
Agora, vamos criar um recurso de Ingress para nosso serviço nginx criado anteriormente. Crie um arquivo chamado ingress-01.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service: 
            name: nginx
            port:
              number: 5000
```
Criado o arquivo, aplique
```bash
kubectl apply -f ingress-01.yaml
```
Verificar ingress
```bash
kubectl get ingress
```
Para detalhar, use
```bash
kubectl describe ingress ingress-01
```
Para pegar o ip, use o seguinte comando
```bash
kubectl get ingress ingress-01 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
caso seja um cluster gerenciado por algum provedor de nuvem, utilizar o seguinte comando.
```bash
kubectl get ingress ingress-1 -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Um cluster EKS, AKS, GCP, etc, o Ingress Controller cria um LoadBalancer, e o endereço IP do LoadBalancer será o endereço IP do seu Ingress
para teste o hostname ou o LB, utilizar o seguinte comando.
```bash
curl ENDEREÇO_DO_INGRESS/nginx
```
