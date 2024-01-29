# Desafio Kubernetes - Configuração do Nginx com HTTPS

Neste desafio, o objetivo é configurar o Nginx com HTTPS em um cluster Kubernetes. Isso envolve a criação e manipulação de Secrets e ConfigMaps para demonstrar compreensão dos conceitos e habilidades práticas em Kubernetes.

## Instruções

**IMPORTANTE:** Todos os arquivos devem ser criados no diretório `/root/manifests`, incluindo o arquivo `nginx.conf`, o certificado (`nginx.crt`), e a chave privada (`nginx.key`).

1. 🏗️ Crie um Secret no Kubernetes que contenha o certificado e a chave privada para o HTTPS. Verifique se o Secret foi criado corretamente e se o tipo está correto, usando o seguinte comando:

    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /root/manifests/nginx.key -out /root/manifests/nginx.crt
    kubectl create secret tls nginx-secret --cert=/root/manifests/nginx.crt --key=/root/manifests/nginx.key
    ```

2. 🎯 Crie um ConfigMap com a configuração do Nginx. Certifique-se de que a configuração está correta e de que o ConfigMap está devidamente configurado.

3. 💼 Verifique se o Nginx está rodando e se seu status é "Running".

4. 🕵️‍♀️ Certifique-se de que o HTTPS está funcionando corretamente. Execute o seguinte comando para verificar:

    ```bash
    curl -k https://localhost:<PORTA>
    ```

5. 📝 Verifique o certificado do Nginx e certifique-se de que está correto.

6. 💡 Verifique se o Service é do tipo NodePort na porta 32400 e está redirecionando para a porta 443 do container.

7. 📝 Verifique se o Nginx está rodando na porta 443.

8. 🎯 O nome do Pod precisa ser `nginx-https`.

9. 🎯 O nome do Service precisa ser `nginx-service`.

10. 🎯 O nome do ConfigMap precisa ser `nginx-config`.

11. 🎯 O nome do Secret precisa ser `nginx-secret`.

12. 🎯 O certificado e a chave privada precisam estar no diretório `/root/manifests`.

13. 🎯 O arquivo de configuração do Nginx precisa estar no diretório `/root/manifests` e se chamar `nginx.conf`.

14. 🎯 O certificado precisa se chamar `nginx.crt`.

15. 🎯 A chave privada precisa se chamar `nginx.key`.

16. 🎯 O manifesto do Pod precisa se chamar `nginx-https-pod.yaml`.

# Iniciando

Para referência futura, abaixo estão os comandos Kubernetes para a criação do certificado e do Secret:

# 1. Criar certificado
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /root/manifests/nginx.key -out /root/manifests/nginx.crt
```

# 2. info
```bash
AU br
full name: ce
state province: name ce
localy name: fortaleza
Organization name: pick
Organization Unit: TI
```

# 3. Criar Secret
```bash
kubectl create secret tls nginx-secret --cert=/root/manifests/nginx.crt --key=/root/manifests/nginx.key
```

# 4. Verificar secret
```bash
k get secret
k get secret nginx-secret -o yaml
```

# 5. Criar nginx.conf
```bash
events { } # configuração de eventos

http { # configuração do protocolo HTTP, que é o protocolo que o Nginx vai usar
  server { # configuração do servidor
    listen 80; # porta que o Nginx vai escutar
    listen 443 ssl; # porta que o Nginx vai escutar para HTTPS e passando o parâmetro ssl para habilitar o HTTPS
    
    ssl_certificate /etc/nginx/tls/nginx.crt; # caminho do certificado TLS
    ssl_certificate_key /etc/nginx/tls/nginx.key; # caminho da chave privada

    location / { # configuração da rota /
      return 200 'Bem-vindo ao Nginx!\n'; # retorna o código 200 e a mensagem Bem-vindo ao Nginx!
      add_header Content-Type text/plain; # adiciona o header Content-Type com o valor text/plain
    } 
  }
}
```

# 6. Criar Configmap

```bash
kubectl create configmap nginx-config --from-file=nginx.conf
```

ou 

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events { }

    http {
      server {
        listen 80;
        listen 443 ssl;

        ssl_certificate /etc/nginx/tls/nginx.crt;
        ssl_certificate_key /etc/nginx/tls/nginx.key;

        location / {
          return 200 'Bem-vindo ao Nginx!\n';
          add_header Content-Type text/plain;
        }
      }
    }
```

# 7. Verificar Configmap
nginx-config.yaml
```bash
k get configmap
k describe configmap nginx-config
k get configmap -o yaml
k get configmap nginx-config -o > configmap.yaml
```
# 8. Utilizando configmap dentro do pod
nginx-https-pod.yaml
```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx-https
  labels:
    app: nginx-https
spec:
  containers:
  - name: nginx-https
    image: nginx
    ports:
    - containerPort: 80
    - containerPort: 443
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: nginx-tls
      mountPath: /etc/nginx/tls
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
  - name: nginx-tls
    secret:
      secretName: nginx-secret
      items:
        - key: tls.crt
          path: nginx.crt
        - key: tls.key
          path: nginx.key
```

# 9. Verificar pod
```bash
k describe pods nginx-service
```

# 10. Criar Service
nginx-service.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-https
  ports:
    - protocol: TCP
      port: 32400  # Porta NodePort
      targetPort: 443  # Porta do Pod
  type: NodePort
```
# 11. Expondo nginx
```bash
k expose pods nginx-service
k get svc
k port-forward services/nginx-service 4443:443
curl -k https://localhost:4443
```