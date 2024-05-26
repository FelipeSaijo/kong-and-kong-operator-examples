# Kong Operator

Kong Gateway é um API Gateway que controla o trafego de requisições e redireciona as requisições para um determinado serviço, também podemos definir em cada path o metodo de requisições aceitadas, configurar plugins para as rotas determiando por exemplo a quantidade de requisições feitas em um path por hora. O operator nos possibilita criar rotas, plugins, etc. atraves de manifestos kubernetes. Para saber mais [kong doc](https://docs.konghq.com/gateway/latest/).

## Docs de referencia

- [Kong operator](https://docs.konghq.com/gateway-operator/latest/)
- [Configurações](https://tech.aufomm.com/explore-kubernetes-gateway-api-with-kong-ingress-controller/)

## Installing kong operator

- Adicione o repositorio do kong na sua maquina e atualize os repositorios:
``` bash
helm repo add kong https://charts.konghq.com && \
helm repo update
```

- Use o seguinte comando para instalar/atualizar o kong gateway operator:
``` bash
helm upgrade --install kong-operator kong/gateway-operator -f kong-operator.yaml -n kong-system --create-namespace
```

## Config Kong dataplane & controlplane

O `GatewayConfiguration` irá criar dois pod, o dataplane responsavel por receber e redirecionar as requisições das rotas configuradas e o controlplane responsavel por capturar as configurações de rotas, plugins, etc. e enviar para o dataplane.
- Edite o arquivo `GatewayConfiguration.yaml` caso deseje criar um LoadBalancer:
``` yaml
  dataPlaneOptions:
    deployment:
      podTemplateSpec:
        spec:
          containers:
          - name: proxy
            image: kong:3.6.1
            readinessProbe:
              initialDelaySeconds: 1
              periodSeconds: 1
## Create dataplane with service ClusterIP
    network:
      services:
        ingress:
          annotations: {}
          type: ClusterIP
## Create dataplane with service LoadBalancer
#    network:
#      services:
#        ingress:
#          annotations: {}
```
> [!NOTE]
> Os pods só será criado após criar o Gateway.

- Execute o seguinte comando para criar o `GatewayConfiguration`:
``` bash
kubectl apply -f GatewayConfiguration.yaml
```

- Execute o seguinte comando para criar o Gateway Class(Ingress Class):
``` bash
kubectl apply -f GatewayClass.yaml
```

- Se o service do dataplane foi criado como type LoadBalancer adicione o seu DNS no arquivo `Gateway.yaml`: 
``` yaml
spec:
  gatewayClassName: kong
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
## Add DNS ingress, use if DataPlane service is type LoadBalancer
#    - name: https
#      hostname: test.com.br
#      port: 443
#      protocol: HTTPS
#      allowedRoutes:
#        namespaces:
#          from: All
#      tls:
#        mode: Terminate
#        certificateRefs:
#          - name: test-tls-secret
```

- Execute o seguinte comando para criar o Gateway (Ingress):
``` bash
kubectl apply -f Gateway.yaml
```

- Use o seguinte comando para pegar os services do dataplane:
``` bash
kubectl get svc
```

- Crie um port-forwad do service `dataplane-ingress` como o exemplo a baixo:
``` bash
kubectl port-forward service/dataplane-ingress-kong-z7hdf-8kgkq 8080:80 
```

- Acesse a url `http://localhost:8080`, você deverá ver o ouptut similar ao exemplo abaixo:
``` bash
{
  "message":"no Route matched with those values",
  "request_id":"3e346414b1953e6cdbd0b76dfcdcc612"
}
```

## HTTPRoute

- Execute o seguinte comando para criar um deployment e service para o nginx:
``` bash
kubectl apply -f deployment.yaml
```

- A baixo temos um exemplo de uma rota, que ao acessar a URL(Configurada no Gateway.yaml anteriormente) "exemplo.com/" irá direcionar para o Service do nginx dentro do cluster.
``` yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: nginx-route # Nome de identificação do HTTPRoute
  annotations:
    konghq.com/strip-path: 'false' # Se 'true' remove o path e redireciona a requisição para o "/" do backend configurado
spec:
  parentRefs:
    ## Referencia ao Gateway criado anteriormente
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: kong
  rules:
    # Configuração de path, metodos, etc.
    - matches:
        - path:
            type: PathPrefix
            value: /nginx
          method: "GET"
    # Caso deseje adicionar outro método é necessário repetir a estrutura "matches"
    # - matches:
    #     - path:
    #         type: PathPrefix
    #         value: /
    #       method: "POST"
      # Referencia o Service kubernetes do serviço que será roteado a requisição
      backendRefs:
        - name: nginx-service
          kind: Service
          port: 80
```

- Aplique as configurações de rotas:
``` bash
kubectl apply -f routes/nginx-route.yaml
```

- Em um terminal ou navegador acesse a seguinte URL `http://localhost:8080/nginx`, você será direcionado para a página principal do nginx.

## Plugins

- [Lista de Plugins](https://docs.konghq.com/hub/?)

- No exemplo abaixo está sendo criado um plugin do tipo `rate-limiting` e uma rota para acessar o serviço nginx. O Plugin esta definindo que o acesso a URL só pode ser feito 5 vezes por IP. O plugin deve ser referenciado na rota, ele não é automaticamente aplicado a todas as rotas.
``` yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting # Nome de indentificação
plugin: rate-limiting # Nome do plugin
instance_name: rate-limiting-5-hit-hour # Identificação do plugin caso mais de um plugin do mesmo tipo for criado
disabled: false # Configuração para habilitar ou desabilitar o plugin
# Configurações do plugin
config:
  hour: 5
  policy: local
  limit_by: ip
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx
  annotations:
    konghq.com/strip-path: "true"
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: kong
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /nginx
          method: "GET"
      filters:
        # Configuração para aplicar um plugin existente a rota, caso outro plugin seja adicionado repida a estrutura abaixo
        - type: ExtensionRef
          extensionRef:
            group: configuration.konghq.com
            kind: KongPlugin
            name: rate-limiting # Nome de indentificação do plugin criado acima
      backendRefs:
        - name: nginx-service
          kind: Service
          port: 80
```

- Execute o seguinte comando para criar o plugin:
``` bash
kubectl apply -f plugins/rate-limiting.yaml
```

- Acesse a seguinte URL `http://localhost:8080/nginx` 6 vezes, na sétima vez você deverá ver uma mensagem similar como o exemplo abaixo significando que o plugin foi configurado corretamente:
``` bash
{
  "message":"API rate limit exceeded",
  "request_id":"724f4605e6caa6699338fa4c7508eb77"
}
```