# Kong Operator

Kong Gateway é um API Gateway que controla o trafego de requisições e redireciona as requisições para um determinado serviço, também podemos definir em cada path o metodo de requisições aceitadas, configurar plugins para as rotas determiando por exemplo a quantidade de requisições feitas em um path por hora. O operator nos possibilita criar rotas, plugins, etc. atraves de manifestos kubernetes. Para saber mais [kong doc](https://docs.konghq.com/gateway/latest/).

## Docs de referencia

- [Kong operator](https://docs.konghq.com/gateway-operator/latest/)
- [Configurações](https://tech.aufomm.com/explore-kubernetes-gateway-api-with-kong-ingress-controller/)

## Instalação

- Adicione o repositorio do kong na sua maquina e atualize os repositorios:
``` bash
helm repo add kong https://charts.konghq.com && \
helm repo update
```

- Use o seguinte comando para instalar/atualizar o kong gateway operator:
``` bash
helm upgrade --install kong-operator kong/gateway-operator -f kong-operator.yaml -n kong-system --create-namespace
```

- Aplique os seguinte templates para criar o gateway (ingress), gatewayclass (ingressClass) e gatewayconfiguration (2 pods que serão responsaveis por receber e redirecionar as requisições):
``` bash
kubectl apply -f GatewayConfiguration.yaml && \
kubectl apply -f GatewayClass.yaml && \
kubectl apply -f Gateway.yaml
```

- Aplique as configurações de rotas:
``` bash
kubectl apply -f routes
```

## HTTPRoute

- A baixo temos um exemplo de uma rota, que ao acessar a URL(Configurada no Gateway.yaml anteriormente) "exemplo.com/" irá direcionar para o Service do nginx dentro do cluster.
``` yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: nginx # Nome de identificação do HTTPRoute
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
            value: /
          method: "GET"
    # Caso deseje adicionar outro método é necessário repetir a estrutura "matches"
    - matches:
        - path:
            type: PathPrefix
            value: /
          method: "POST"
      # Referencia o Service kubernetes do serviço que será roteado a requisição
      backendRefs:
        - name: nginx-service
          kind: Service
          port: 8080
```

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
            value: /test
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
          port: 8080
```