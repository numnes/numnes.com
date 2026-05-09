---
title: Deployer
author: Matheus Nunes
pubDatetime: 2026-05-06T01:06:31Z
slug: deployer
featured: false
draft: true
tags:
  - TypeScript
  - Deploy
  - devops
  - deployer
description: Uma pequena solução para a gestão de instancias de teste auto hosteadas
---

![](https://res.cloudinary.com/nunes/image/upload/v1778291231/Obsidian/8-go-wrong-da386481824319a4c33957f89459d698_e9dr2y.jpg)

## Prólogo

Task feita, PR aberto, PR aprovado no code review, agora é a hora de testar... Tá, mas como?

Nas empresas que eu já trabalhei esse era sempre um problema. Como garantir que o código feito na tarefa funciona realmente? Normalmente, empresas pequenas ou até mesmo de porte grande em parâmetros de interior do Brasil, tem infraestrutura e verba limitada, só existem dois ambientes, um de desenvolvimento e outro de produção, daí a regra vira subir tudo que foi aprovado no code review (se ele existir) para o ambiente de dev, onde um amontoado de tarefas são testadas e se algo que funcionava não funciona mais, ninguém sabe qual tarefa quebrou tudo.
Mas também existe uma outra abordagem, tão emocionante quanto, fazer o QA puxar o código de cada tarefa para a branch local, e então, testar na própria máquina se o código que vai rodar em um datacenter funciona no seu I5 de 10ª geração e 8Gb de memória.

É claro que a forma ideal de testar um software varia conforme o formato do produto que está sendo desenvolvido, a configuração da equipe e como os prazos e entregas são administrados, mas em geral, para produtos que rodam na web e tem pacotes de entrega bem definidos e com prazo, o ideal é garantir que cada funcionalidade desenvolvida funcione por si só, e nesse caso o melhor cenário para os testes é bem claro.
Deveria haver um ambiente próprio para cada tarefa, com a maior similaridade possível com o ambiente de produção e com o máximo de isolamento de recursos entre cada um.

Exemplificando:
![](https://res.cloudinary.com/nunes/image/upload/v1778296147/Obsidian/image_otiikn.png)

Para leitores que trabalham em empresas com um fluxo de teste bem estruturado, pode parecer que eu estou só vomitando obviedades, mas a maior parte dos produtos web que são produzidos por pequenas empresas (que são a maior parte dos produtos web), não tem um fluxo de teste como esse.

Existem algumas soluções que entregam um fluxo como esse já automatizado, como a Vercel e o Amplify da AWS, mas em empresas que não tem recursos para contratar um serviço ou que tem sua infraestrutura presa a servidores próprios ou de baixo custo, isso é algo complicado de implementar.

## Mas por que esse papo?

Atualmente estou trabalhando para uma empresa que tem um fluxo bem estruturado como esse, cada tarefa que é aberta e enviada para teste tem uma instância própria e é em cima dela que a equipe responsável realiza os testes. Cada tarefa que é mandada para o ambiente de homologação, foi testada e está funcionando como deveria, quando vi pela primeira vez como isso funcionava bem, pensei em projetos que trabalhei antes e todo tempo perdido tentando encontrar qual tarefa quebrou a homologação ou ensinando para o tester como resolver o problema que o código estava dando na sua máquina local.
Esta estrutura que foi feita "na unha", e tem vários gaps que ainda travam o fluxo mas acredito que esse seja o cenário ideal para testes de qualquer produto web.

Pensei então em desenvolver uma solução que pudesse simplificar a implantação de um ambiente assim, em qualquer máquina e que facilitasse a visualização e orquestração das instâncias de teste.

## Deployer

Decidi documentar aqui o passo a passo do planejamento e desenvolvimento dessa ideia, que chamei de Deployer (acho que define bem o conceito), então depois de todo este texto, segue um devlog de como foi implementar essa ideia.
Todo o código desenvolvido está nesse repositório.

Primeiramente, pensei em estruturar a parte fundamental da ideia: O fluxo de deploy.
Então preciso pensar nas peças que vão compor esse fluxo:

- **Github Actions**: Vou usar o Github Actions para definir uma action que vai identificar a abertura de um pr e disparar a chamada para a máquina onde o deploy vai acontecer.
- **Api**: Preciso de uma api que vai receber o request de deploy da branch, feito pela action e depois chamar a parte responsável por executar o deploy em si.
- **Core**: Decidi chamar assim uma coleção de scripts shell que vão fazer toda a parte bruta da tarefa, desde puxar a branch para a máquina, disparar a build, colocar o projeto para rodar e depois configurar a rota no proxi para rotear a url onde a instaância especifica da branch vai estar disponível.

A ideia geral para o comportamento de provisionamento é essa:
![](https://res.cloudinary.com/nunes/image/upload/v1778299234/Obsidian/image_udlx7s.png)

### API

Já que a parte da action é simples e ela precisa de uma api para chamar, vou começar com a api. Vou usar o NestJs pois é o framework que tenho mais familiaridade, e usei o Postgres para armazenar os dados das entidades que vou detalhar logo mais.
Vou precisar definir alguns módulos na minha api.

- **api-keys**: Como a api vai estar pública, não é bom deixar ela disponível para que requests sejam feitas sem autenticação, então preciso gerar e administrar chaves de api que a action vai usar para se autenticar.

  **Endpoints**:
  - POST /api-keys: Para gerar a chave de api.
  - GET /api-keys: Para gerenciar as chaves de api geradas.
  - DELETE /api-keys: Para revogar uma chave.

- **auth**: Preciso de alguns endpoints para criar usuários, já que não quero que qualquer um gere uma chave de api e quero também criar uma interface web logada para ver as minhas instâncias.

  **Endpoints**:
  - POST /auth/login: Para fazer o login e obter um token.
  - POST /auth/register: Para criar meus usuários.

- **projects**: A entidade deste módulo vai guardar informações sobre o repósitorio, como qual a url dele no Github e qual domínio as instâncias dele vão usar (vai ficar mais claro isso depois). Não vou detalhar aqui os seus endpoints.

- **instances**: Esta entidade vai guardar as informações sobre a instancia que está rodando, como qual o seu path atribuído, qual a porta ele está usando na máquina host etc.

- **deploy**: Este vai ser o módulo com os endpoints que vão chamar o deploy de uma instancia. A autenticação nos seus endpoints é feita com o token de usuário ou com a api-key gerada no modulo de api-keys.

  **Endpoints:**
  - POST /deploy: Dispara a criação de uma instancia.
  - POST /deploy/destroy: Dispara a remoção de uma instância.

O código da api está na pasta server no repositório do post, ele está bem comentado então fica mais fácil olhar por lá os detalhes da implementação destes módulos básicos.

### GITHUB ACTIONS

A função de actions do github permite definir ações que serão disparadas em momentos específicos conforme gatilhos disparados por iterações com o repositório. As ações são executadas na cloud do Github e executam os passos definidos no documento .yaml que descreve a action.

Quero dois comportamentos da action que vai disparar a ferramenta, ao criar um PR na branch EXP-1234, por exemplo, quero que ele faça uma request para a minha api solicitando o levantamento da instancia de teste da branch, e quando o pr for mergeado quero que ele faça uma request requisitando a derrubada.
Para manter a segurança das credenciais, precisamos definir os secrets que terão a URL pública da api e a api-key que geramos para nos autenticar. A definição da action fica basicamente esta:

```yaml
name: Deploy

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - master
      - homologation

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy na ferramenta
        env:
          DEPLOYER_API_URL: ${{ secrets.DEPLOYER_API_URL }}
          DEPLOYER_API_KEY: ${{ secrets.DEPLOYER_API_KEY }}
          PROJECT_SLUG: ${{ github.event.repository.name }}
        run: |
          if [ -z "$DEPLOYER_API_URL" ] || [ -z "$DEPLOYER_API_KEY" ]; then
            echo "Defina os secrets DEPLOYER_API_URL e DEPLOYER_API_KEY."
            exit 1
          fi
          #BRANCH="${{ github.head_ref }}"
          BRANCH="${{ github.ref_name }}"
          curl -fsS -X POST "${DEPLOYER_API_URL%/}/deploy" \
            -H "Content-Type: application/json" \
            -H "X-Deployer-Api-Key: ${DEPLOYER_API_KEY}" \
            -d "{\"project\":\"${PROJECT_SLUG}\",\"branch\":\"${BRANCH}\"}"

  teardown-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Remover preview
        env:
          DEPLOYER_API_URL: ${{ secrets.DEPLOYER_API_URL }}
          DEPLOYER_API_KEY: ${{ secrets.DEPLOYER_API_KEY }}
          PROJECT_SLUG: ${{ github.event.repository.name }}

        run: |
          if [ -z "$DEPLOYER_API_URL" ] || [ -z "$DEPLOYER_API_KEY" ]; then
            echo "Defina os secrets DEPLOYER_API_URL e DEPLOYER_API_KEY."
            exit 1
          fi
          #BRANCH="${{ github.head_ref }}"
          BRANCH="${{ github.ref_name }}"
          curl -fsS -X POST "${DEPLOYER_API_URL%/}/deploy/destroy" \
            -H "Content-Type: application/json" \
            -H "X-Deployer-Api-Key: ${DEPLOYER_API_KEY}" \
            -d "{\"project\":\"${PROJECT_SLUG}\",\"branch\":\"${BRANCH}\"}"
```

Agora já posso testar se a parte de disparar o deploy está funcionando, então registrei o meu projeto no modulo de projects, setando a url do github e o nome do repositório.

Para testar localmente preciso expor a minha api, então estou usando o serviço de tunneis da cloudflare, que acho bem prático.
