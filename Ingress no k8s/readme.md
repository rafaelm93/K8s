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

