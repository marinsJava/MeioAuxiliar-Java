# Guia Definitivo de Kubernetes

> Orquestração de containers do conceito ao cluster rodando, com manifestos voltados a aplicações Spring Boot + PostgreSQL. Versão de referência: **Kubernetes 1.36** (junho/2026).

Docker resolve "como empacotar e rodar **um** container". Kubernetes (abreviado **K8s** — o 8 são as letras entre o "K" e o "s") resolve "como rodar **centenas** deles em produção, com escala automática, auto-recuperação, balanceamento de carga e zero downtime em deploys". É o padrão de mercado para orquestração.

---

## Índice

- [Quando você precisa (e quando não precisa) de Kubernetes](#quando-você-precisa)
- [Arquitetura do cluster](#arquitetura-do-cluster)
- [Os objetos fundamentais](#os-objetos-fundamentais)
- [kubectl: a ferramenta de linha de comando](#kubectl)
- [Subindo um cluster local para estudar](#subindo-um-cluster-local)
- [Deployment de uma aplicação Spring Boot](#deployment-de-uma-aplicação-spring-boot)
- [Service: expondo a aplicação](#service-expondo-a-aplicação)
- [ConfigMap e Secret: configuração e segredos](#configmap-e-secret)
- [Health checks com Spring Boot Actuator](#health-checks-com-spring-boot-actuator)
- [Escala automática (HPA)](#escala-automática-hpa)
- [E o banco de dados? StatefulSet vs. managed](#e-o-banco-de-dados)
- [Helm: o gerenciador de pacotes](#helm)
- [Referências](#referências)

---

## Quando você precisa

Sendo honesto desde o começo: **Kubernetes é poderoso e complexo, e a maioria dos projetos pequenos não precisa dele.** Para uma aplicação única ou poucos serviços, `docker compose` num servidor, ou uma plataforma gerenciada (Railway, Render, Fly.io, App Runner), resolve com uma fração do esforço.

K8s passa a valer a pena quando você tem: muitos serviços, necessidade real de escalar sob demanda, exigência de alta disponibilidade, deploys frequentes sem downtime, ou um time de plataforma para operar. Adotá-lo cedo demais é trocar um problema que você tem por dez que você não tinha. Dito isso, entender Kubernetes é uma habilidade valiosa — só aplique com critério.

---

## Arquitetura do cluster

Um cluster Kubernetes tem dois tipos de máquina (nós): o **control plane** (o cérebro, que decide) e os **worker nodes** (os músculos, que executam suas aplicações).

```
┌─────────────────── CONTROL PLANE (cérebro) ───────────────────┐
│  ┌─────────────┐  ┌──────────┐  ┌────────────┐  ┌───────────┐ │
│  │ API Server  │  │   etcd   │  │ Scheduler  │  │Controller │ │
│  │ (a porta de │  │ (o banco │  │ (decide em │  │  Manager  │ │
│  │  entrada)   │  │  de dados│  │ qual node) │  │(reconcilia│ │
│  └─────────────┘  │ do estado│  └────────────┘  │  estado)  │ │
│                   └──────────┘                  └───────────┘ │
└───────────────────────────────┬───────────────────────────────┘
                                 │ (kubectl fala com o API Server)
        ┌────────────────────────┼────────────────────────┐
┌───────▼────────┐      ┌────────▼───────┐      ┌──────────▼─────┐
│  WORKER NODE   │      │  WORKER NODE   │      │  WORKER NODE   │
│ ┌────────────┐ │      │ ┌────────────┐ │      │ ┌────────────┐ │
│ │  kubelet   │ │      │ │  kubelet   │ │      │ │  kubelet   │ │
│ │ kube-proxy │ │      │ │ kube-proxy │ │      │ │ kube-proxy │ │
│ │ [ Pods ]   │ │      │ │ [ Pods ]   │ │      │ │ [ Pods ]   │ │
│ └────────────┘ │      │ └────────────┘ │      │ └────────────┘ │
└────────────────┘      └────────────────┘      └────────────────┘
```

Componentes do **control plane**:
- **API Server** — a porta de entrada de tudo. Toda comunicação (sua e dos componentes internos) passa por ele.
- **etcd** — banco de dados chave-valor que guarda o estado completo do cluster. É a fonte da verdade.
- **Scheduler** — decide em qual node cada Pod vai rodar, com base em recursos disponíveis.
- **Controller Manager** — roda os *controllers* que mantêm o estado real igual ao desejado (o coração do modelo declarativo).

Componentes dos **worker nodes**:
- **kubelet** — o agente que conversa com o control plane e garante que os containers do node estejam rodando.
- **kube-proxy** — cuida das regras de rede e roteamento.
- **Container runtime** — quem de fato roda os containers (containerd, hoje o padrão).

> A ideia-mãe do Kubernetes é o **modelo declarativo**: você descreve o *estado desejado* ("quero 3 réplicas dessa app rodando"), e o cluster trabalha continuamente para fazer a realidade bater com isso. Se um Pod morre, um controller cria outro. Você não dá ordens passo a passo; você declara o destino.

---

## Os objetos fundamentais

| Objeto | O que faz |
|---|---|
| **Pod** | A menor unidade. Envolve um (ou mais) containers que compartilham rede e armazenamento. Efêmero. |
| **ReplicaSet** | Garante que N réplicas de um Pod estejam sempre rodando. |
| **Deployment** | Gerencia ReplicaSets e cuida de atualizações sem downtime (rolling updates) e rollback. É o que você usa no dia a dia. |
| **Service** | Endereço de rede estável para um conjunto de Pods (que vêm e vão). Faz balanceamento de carga. |
| **Ingress** | Roteamento HTTP/HTTPS de fora para dentro do cluster (com domínios e TLS). |
| **ConfigMap** | Armazena configuração não-sensível. |
| **Secret** | Armazena dados sensíveis (senhas, tokens). |
| **Namespace** | Divisão lógica do cluster (ex.: `dev`, `prod`). |
| **PersistentVolumeClaim (PVC)** | Pedido de armazenamento persistente. |
| **StatefulSet** | Como o Deployment, mas para apps com estado e identidade fixa (ex.: bancos). |

> Você quase nunca cria Pods diretamente. Você cria um **Deployment**, que cria um **ReplicaSet**, que cria os **Pods**. Cada nível adiciona uma garantia.

---

## kubectl

`kubectl` é a CLI que conversa com o cluster. Comandos essenciais:

```bash
# Aplicar/remover manifestos (modo declarativo — o jeito certo)
kubectl apply -f deployment.yaml      # cria ou atualiza
kubectl delete -f deployment.yaml     # remove

# Inspecionar
kubectl get pods                      # lista pods
kubectl get all                       # lista tudo do namespace
kubectl describe pod <nome>           # detalhes e eventos (ótimo para debug)
kubectl logs -f <pod>                 # acompanha logs
kubectl logs <pod> --previous         # logs do container anterior (após um crash)

# Interagir
kubectl exec -it <pod> -- sh          # shell dentro do pod
kubectl port-forward <pod> 8080:8080  # acessa o pod localmente sem expor

# Escala e rollout
kubectl scale deployment/api --replicas=5
kubectl rollout status deployment/api
kubectl rollout undo deployment/api   # rollback para a versão anterior

# Contexto e namespace
kubectl config get-contexts
kubectl get pods -n meu-namespace
```

> Prefira sempre o modo **declarativo** (`kubectl apply -f arquivo.yaml`) ao imperativo (`kubectl create ...`). Os manifestos YAML vão para o Git e viram a fonte da verdade — base do GitOps.

---

## Subindo um cluster local

Para estudar, você não precisa de servidores. Há três opções leves que sobem um cluster na sua máquina:

```bash
# minikube — o mais clássico e didático
minikube start
minikube dashboard          # abre o painel web

# kind (Kubernetes IN Docker) — cluster roda dentro de containers Docker
kind create cluster --name estudo

# k3d — embrulha o k3s (distribuição leve), muito rápido
k3d cluster create estudo
```

> Para quem já usa Docker Desktop, há um Kubernetes embutido que você habilita nas configurações com um clique. É o caminho mais rápido para começar no Windows/WSL2.

---

## Deployment de uma aplicação Spring Boot

Aqui tudo se junta. Este manifesto declara "quero 3 réplicas da minha API rodando":

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-api
  labels:
    app: spring-api
spec:
  replicas: 3                      # 3 cópias para disponibilidade e carga
  selector:
    matchLabels:
      app: spring-api
  template:
    metadata:
      labels:
        app: spring-api
    spec:
      containers:
        - name: spring-api
          image: ghcr.io/seu-usuario/spring-api:1.0
          ports:
            - containerPort: 8080
          resources:              # limites evitam que um pod sufoque o node
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secret
```

Sempre defina `resources.requests` e `limits`. Sem `requests`, o scheduler não sabe alocar; sem `limits`, um pod com vazamento de memória pode derrubar o node inteiro. Para apps Java, dimensione a memória considerando a heap da JVM **mais** o overhead fora da heap (metaspace, threads).

---

## Service: expondo a aplicação

Pods são efêmeros e mudam de IP. O **Service** dá um endereço estável e balanceia carga entre as réplicas.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-api
spec:
  selector:
    app: spring-api          # conecta a todos os pods com esse label
  ports:
    - port: 80               # porta do Service
      targetPort: 8080       # porta do container
  type: ClusterIP            # acessível só dentro do cluster
```

Tipos de Service: **ClusterIP** (interno, padrão), **NodePort** (expõe numa porta de cada node), **LoadBalancer** (provisiona um balanceador do provedor de nuvem). Para expor HTTP com domínio e TLS de fora, normalmente combina-se um Service `ClusterIP` com um **Ingress**.

---

## ConfigMap e Secret

Nunca coloque configuração nem senha dentro da imagem. Separe-as em objetos próprios.

```yaml
# ConfigMap — configuração não-sensível
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/meuapp"
---
# Secret — dados sensíveis
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
type: Opaque
stringData:                    # stringData aceita texto puro (o cluster codifica)
  SPRING_DATASOURCE_PASSWORD: "senha-super-secreta"
```

> ⚠️ Secrets do Kubernetes são apenas **codificados em Base64**, não criptografados por padrão — qualquer um com acesso pode decodificar. Para produção, use criptografia em repouso no etcd e ferramentas como Sealed Secrets, External Secrets Operator ou Vault. **Nunca** versione Secrets reais no Git.

---

## Health checks com Spring Boot Actuator

O Kubernetes precisa saber se seu pod está vivo e pronto. O Spring Boot Actuator entrega isso de bandeja com endpoints prontos.

Adicione o Actuator e habilite as *probes*:

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health
```

E configure as três sondas no Deployment:

```yaml
          startupProbe:           # dá tempo para a JVM subir antes das outras checarem
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:          # se falhar, o pod é REINICIADO
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 10
          readinessProbe:         # se falhar, o pod é tirado do balanceamento (mas não reiniciado)
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 10
```

A distinção é crucial: **liveness** responde "preciso reiniciar?"; **readiness** responde "posso receber tráfego agora?". O `startupProbe` existe especialmente para apps Java, que demoram a inicializar — ele evita que a liveness mate o pod durante uma partida lenta da JVM.

---

## Escala automática (HPA)

O *Horizontal Pod Autoscaler* ajusta o número de réplicas automaticamente conforme a carga:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # sobe réplicas quando a CPU média passa de 70%
```

Requer o *metrics-server* instalado no cluster. Com isso, sua API cresce em picos e encolhe em vales, sozinha.

---

## E o banco de dados?

A pergunta que sempre surge: "rodo o PostgreSQL no Kubernetes?". A resposta honesta tem duas partes:

**Em produção, na dúvida, use um banco gerenciado** (RDS, Cloud SQL, Azure Database). Rodar banco com estado em K8s exige cuidado real com persistência, backup, failover e atualizações — é trabalho de especialista, e o gerenciado tira esse fardo.

**Para estudo ou ambientes não-críticos**, dá para rodar com **StatefulSet** + **PersistentVolumeClaim** (o StatefulSet garante identidade estável e armazenamento dedicado por réplica, ao contrário do Deployment):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:18
          envFrom:
            - secretRef:
                name: postgres-secret
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:           # cada réplica ganha seu próprio volume persistente
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Helm

Escrever dezenas de YAMLs por mão vira insustentável. O **Helm** é o "gerenciador de pacotes" do Kubernetes: empacota manifestos em *charts* parametrizáveis e versionados.

```bash
# Instalar um PostgreSQL pronto, por exemplo, vira:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install meu-pg bitnami/postgresql

# E sua própria app vira um chart com valores por ambiente:
helm install spring-api ./meu-chart -f values-prod.yaml
```

Helm resolve o problema de reusar e parametrizar manifestos entre ambientes (dev/staging/prod) sem copiar e colar YAML.

---

## Referências

1. **Documentação oficial** — https://kubernetes.io/docs/home/
2. **kubectl Cheat Sheet** — https://kubernetes.io/docs/reference/kubectl/cheatsheet/
3. **Spring Boot — Kubernetes** — https://docs.spring.io/spring-boot/reference/deployment/cloud.html#deployment.cloud.kubernetes
4. **Helm** — https://helm.sh/docs/
5. **Kubernetes the Hard Way** (Kelsey Hightower) — para entender o que está por baixo da abstração: https://github.com/kelseyhightower/kubernetes-the-hard-way

> Pré-requisitos deste guia: estar confortável com containers. Se ainda não estiver, comece pelo [Guia Definitivo de Docker](../docker/guia-definitivo-docker.md). Para o banco, veja o [Guia Definitivo de PostgreSQL](../../bancos-de-dados/postgresql/guia-definitivo-postgresql.md).
