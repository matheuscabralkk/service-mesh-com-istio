# Curso Full Cycle - Service Mesh com Istio 

Documentação do Istio Service Mesh: https://istio.io/latest/docs/

### 1. Instalação 

###### Criar um cluster com k3d, instalar istio no cluster, injetar o sidecar proxy e instalar os addons *prometheus, kiali, jaeger e grafana*.
1. `k3d cluster create -p "8000:30000@loadbalancer" --agents 2`
   Cria um cluster Kubernetes fazendo um bind da porta 8000 da maquina host para a porta 30000 do cluser, e a porta 30000 aponta pro serviço do kubernetes
2. `istioctl install`
Instala istio no cluster recém criado
3. `kubectl label namespace default istio-injection=enabled`
Injeta o sidecar proxy
4. `kubectl apply -f deployment.yaml`
Vai fazer a instalação dos serviços listados no arquivo **deployment.yaml** no kubernetes 
5. `kubectl get po`
Ver os pods rodando
6. Instalar addons (https://github.com/istio/istio/tree/release-1.12/samples/addons) *abrir em modo "raw" cada arquivo para pegar a url corretamente*
   1. `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/grafana.yaml`
   2. `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/jaeger.yaml`
   3. `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/kiali.yaml`
   4. `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/prometheus.yaml`
7. Utilizando os addons
   1. `kubectl get po -n istio-system` Ver os addons instalados
   2. `istioctl dashboard kiali` vai abrir o dashboard do kiali na porta 20001
8. Pra testarmos o tráfego na rede
   1. `while true; do curl http://localhost:8000; echo; sleep 0.5; done;`
   
###### Utilizando Fortio
8. Instalar seguindo a [documentação](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/#adding-a-client)
   1. `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/httpbin/sample-client/fortio-deploy.yaml`
   2. `export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')`
      1. `echo $FORTIO_POD`
   3. `kubectl exec "$FORTIO_POD" -c fortio -- fortio load -c 2 -qps 0 -t 200s -loglevel warning http://nginx-service:8000`
      1. Vamos executar um comando no pod do Fortio pra carregar com 2 conexões concorrentes, fazendo automaticamente a quantidade por segundo, rodando durante 200 segundos, logando se tiver algum warning mandando pro `http://nginx-service:8000`


###### Consistent Hash
Vai utilizar sempre um pod pra cada client (por ip)
1. pra testar: `curl --header "x-user: m" http://nginx-service:8000` -> vai cair sempre em um cluster, se mudar o x-user, pode ser q caia qm outro

###### Testar o Circuit breaker
1. Simular 20 requisiçoes (metade retorna 200 e metade retorna 504):
   `kubectl exec "$FORTIO_POD" -c fortio -- fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://servicex-service`

###### Outros comandos úteis citados na aula
* `k3d cluster delete` deletar os clusters kubernetes
* `kubectl cluster-info` Ver url do cluster 
* `kubectl get nodes` list os nodes do cluster 
* `kubectl get ns` ver os namespaces do kubernetes
* `kubectl get po -n istio-system` ver pods do namespace istio-system
* `kubectl get svc` ver serviços do kubernetes
* `kubectl get svc -n istio-system`  ver serviços do namespace istio-system do kubernetes

###### Configurando addons
* Repositório com uma lista de addons
https://github.com/istio/istio/tree/release-1.12/samples/addons

### 2. Introdução
###### Service Mesh 
- Você Garante a Observabilidade e Rastreamento de Suas Aplicações.
- É uma camada extra adicional junto ao cluster (do kubernetes, apache mesos, consul e nomad)
- Funcionada na camada de rede, não da aplicação
###### Istio
- projeto open-source que implementa service mesh
###### pq preciso de service mesh e/ou istio?
- gerenciamento de tráfego
  - gateways, load balancing, timeout, politicas de retry, circuit breaker, fault injection
- observabilidade
  - métricas, traces distribuídos, logs
- segurança
  - man-in-the-middle, mTLS, AAA (authentication, authorization e audit)
###### Sidecar Proxy
- dinâmica
	- são as entradas e saídas da rede de cada serviço ou pod
	- vai ter as regras de retry
- istioD
	- Configuração do istio
	- control plane
###### Arquitetura Sidecar Proxy
<img src="readme_imgs/arch.svg" alt="drawing" width="700"/>

### 3. Gerenciamento de tráfego
- Ingress Gateway
  - trabalha nos layers 4-6
  - Componente do **Istio** que recebe conexões de fora do cluster (parecido com o **ingress do Kubernetes**)
  - Ligado diretamente com o **Virtual Service** que é uma espécie de **roteador**
  - O **ingress gateway** libera a entrada do trafego e o **Virtual Service** faz o roteamento desse tráfego
  - Recursos do **Virtual Service**
    - Match; Retries; Fault Injection; timeout; Subsets (Destination Roles).
  - **Destination Roles**
    - **Selector**: filtra versão do sistema pra trabalhar;
    - **tipo de Load Balancer**;
    - **Locality**: balanceamento baseado em localidade
    - **Circuit Breaker**;
