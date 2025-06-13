Relatório da Sessão: Implantação e Depuração de Arquitetura GitOps no GKE com Terraform e Argo CD
Data: 13 de Junho de 2025

Objetivo: Implantar uma aplicação web de três camadas (frontend, backend, banco de dados) em um cluster GKE, utilizando automação completa com Terraform para a infraestrutura e Argo CD para a entrega contínua (GitOps).

Fase 1: Provisionamento e Depuração da Infraestrutura Base (Terraform)
Nesta fase, utilizamos um projeto Terraform (pasta infra/) para criar a infraestrutura fundamental no Google Cloud Platform.

Recursos Criados:

Cluster GKE (google_container_cluster).
Node Pool (google_container_node_pool).
Regra de Firewall para acesso via IAP (google_compute_firewall).
Permissões IAM para o IAP.
Principal Desafio Encontrado:

Erro: Error: Provider produced inconsistent final plan na criação da regra de firewall.
Causa Raiz: O Terraform planejava a regra usando o nome simples da rede ("default"), mas o provedor do Google retornava o endereço completo (self_link), causando uma inconsistência.
Solução Definitiva Implementada: Utilizamos uma fonte de dados (data "google_compute_network") para buscar a rede pelo nome "default" e, em seguida, usamos o atributo .self_link no recurso do firewall. Isso garante que o plano do Terraform já contenha o endereço completo, eliminando a ambiguidade.
Fase 2: Estruturação dos Projetos e Automação da Aplicação (Terraform + Kubernetes)
Nesta fase, separamos a lógica em dois projetos Terraform distintos (infra e k8s) para seguir as melhores práticas de IaC.

Conceitos Implementados:
Separação de Responsabilidades: O projeto infra gerencia o cluster, e o projeto k8s gerencia os recursos dentro dele.
Comunicação entre Projetos: Utilizamos o data "terraform_remote_state" na pasta k8s para ler os outputs (como o endpoint do cluster) do estado do projeto infra, criando uma ponte segura e automatizada.
Conversão de Manifestos: Todos os arquivos YAML do Kubernetes foram convertidos para recursos HCL do Terraform (ex: kubernetes_deployment, kubernetes_stateful_set, etc.).
Gerenciamento de Segredos: Substituímos ConfigMaps que continham senhas por recursos kubernetes_secret, e ajustamos os Deployments/StatefulSets para lerem de ambas as fontes (configMapRef e secretRef).
Volumes e Permissões: Depuramos um erro de Permission Denied em volumes persistentes, implementando a solução com initContainers no StatefulSet para executar chown e corrigir as permissões antes da inicialização do contêiner principal.
Desafios de Rede: Depuramos a conectividade entre o frontend e o backend, descobrindo que o ConfigMap do portal precisava do IP público do LoadBalancer do backend. A solução foi fazer essa referência de forma dinâmica no Terraform: urlnoticias = "http://${kubernetes_service.sistema_noticias_lb.status[0].load_balancer[0].ingress[0].ip}".
Fase 3: Implementação de GitOps com Argo CD
Nesta fase final, automatizamos o processo de entrega contínua.

Instalação do Argo CD via Terraform:

Utilizamos o provedor helm do Terraform para instalar o Argo CD de forma automatizada.
Depuramos uma série de erros CrashLoopBackOff e API did not recognize... nos pods do Argo CD.
Causa Raiz: Uma condição de corrida onde o Terraform tentava criar recursos do Argo CD (Application) antes que as definições desses recursos (CRDs) estivessem prontas no cluster.
Solução Definitiva Implementada: Alteramos o Terraform para:
Baixar os manifestos das CRDs essenciais (Application, AppProject, etc.) usando o provedor http.
Aplicar essas CRDs explicitamente com kubernetes_manifest, usando o bloco wait para garantir que fossem estabelecidas.
Instalar o chart do Helm com a opção crds.install=false.
Desabilitar componentes opcionais (dex, applicationset) para garantir a estabilidade da instalação.
Configuração do Fluxo GitOps:

Criamos um novo repositório Git para conter os manifestos YAML (a "fonte da verdade").
Configuramos o Terraform para criar um recurso Application ("App of Apps") que aponta para o repositório Git.
Depuramos o erro final de authentication required ao tornar o repositório Git público para este ambiente de laboratório.
Estado Final da Arquitetura
O resultado é uma arquitetura de nível profissional, totalmente automatizada e reproduzível:

Infraestrutura Gerenciada: Um cluster GKE e seus componentes são criados e gerenciados pelo Terraform no projeto infra.
Plataforma Gerenciada: A ferramenta de GitOps, Argo CD, é instalada e configurada automaticamente pelo Terraform no projeto k8s.
Aplicações Gerenciadas por GitOps: Todas as aplicações (banco, backend, frontend) são definidas em manifestos YAML em um repositório Git. O Argo CD garante que o estado do cluster sempre corresponda ao que está no Git.
Para implantar todo o ambiente do zero, o processo é:

terraform apply na pasta infra.
entrar na pasta k8s
comentar o arquivo argocd-apps.tf
terraform apply no argocd.tf
descomentar e dar terraform apply no argocd-apps.tf


Para atualizar uma aplicação, o processo é:

Fazer uma alteração no manifesto YAML.
git push. O Argo CD cuida do resto.











Deep Research

Canvas

