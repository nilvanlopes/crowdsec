# CrowdSec Nesta Stack

Este diretório contém a configuração do CrowdSec para uma stack Docker Swarm integrada ao Traefik. O objetivo é:

- ler os logs do Traefik
- detectar tráfego malicioso
- expor um bouncer HTTP que o Traefik consulta via `forwardAuth`
- bloquear IPs maliciosos antes que o tráfego chegue aos serviços publicados

O fluxo nesta stack é:

1. o Traefik grava logs no volume `traefik_traefik-logs`
2. o container `crowdsec` monta esse volume em modo leitura
3. o arquivo `acquis.yml` instrui o CrowdSec a ler `/var/log/traefik/*`
4. o serviço `bouncer-traefik` consulta a API do CrowdSec
5. o Traefik usa o middleware `crowdsec@file` para chamar o bouncer em `http://bouncer-traefik:8080/api/v1/forwardAuth`

## Arquivos Deste Diretório

- [`docker-compose.yml`](/mnt/d/docker/crowdsec/docker-compose.yml): define os serviços `crowdsec` e `bouncer-traefik`
- [`config/acquis.yml`](/mnt/d/docker/crowdsec/config/acquis.yml): informa ao CrowdSec quais logs devem ser analisados
- [`.env.example`](/mnt/d/docker/crowdsec/.env.example): modelo das variáveis necessárias
- [`.env`](/mnt/d/docker/crowdsec/.env): arquivo local com a chave do bouncer

## Dependências

Antes de subir o CrowdSec, estes pré-requisitos precisam existir:

1. Docker Swarm inicializado no host
2. rede overlay `traefik-public`
3. stack do Traefik já implantada
4. volume externo `traefik_traefik-logs` existente
5. config Docker Swarm `crowdsec_acquis` criada a partir de `config/acquis.yml`

Sem isso, o deploy falha ou sobe parcialmente.

## Onde o CrowdSec Se Integra Com o Traefik

Nesta stack, a integração está dividida em dois pontos:

- o Traefik grava logs no volume `traefik-logs`, publicado no stack como `traefik_traefik-logs`
- o middleware dinâmico do Traefik fica em [`traefik/infra/traefik/dynamic/crowdsec.yml`](/mnt/d/docker/traefik/infra/traefik/dynamic/crowdsec.yml)

O middleware configurado ali usa:

- `forwardAuth`
- endpoint `http://bouncer-traefik:8080/api/v1/forwardAuth`

Para um router usar o CrowdSec, ele precisa referenciar o middleware `crowdsec@file`, como já acontece em alguns serviços deste repositório.

## Passo A Passo Completo

### 1. Garanta que o Swarm está ativo

Valide o estado do Swarm:

```bash
docker info | rg "Swarm:"
```

Se necessário:

```bash
docker swarm init
```

### 2. Suba o Traefik antes do CrowdSec

O CrowdSec depende dos logs produzidos pelo Traefik. Sem isso, o volume externo `traefik_traefik-logs` não existirá.

Pelo `makefile`, o caminho esperado é:

```bash
make deploy-traefik
```

Ou, se for a primeira implantação completa do ambiente:

```bash
make deploy
```

### 3. Garanta que a rede `traefik-public` existe

O `make deploy-traefik` já cuida disso, mas você pode validar manualmente:

```bash
docker network ls | rg "traefik-public"
```

Se precisar criar manualmente:

```bash
docker network create --driver=overlay --attachable traefik-public
```

### 4. Crie a config externa `crowdsec_acquis`

O arquivo [`docker-compose.yml`](/mnt/d/docker/crowdsec/docker-compose.yml) declara `crowdsec_acquis` como config externa. Isso significa que ela precisa existir no Swarm antes do deploy:

```bash
docker config create crowdsec_acquis crowdsec/config/acquis.yml
```

Validação:

```bash
docker config ls | rg "crowdsec_acquis"
```

Se a config já existir e você alterar `config/acquis.yml`, o caminho mais seguro é:

1. remover a config antiga
2. recriá-la
3. redeployar o stack

Exemplo:

```bash
docker config rm crowdsec_acquis
docker config create crowdsec_acquis crowdsec/config/acquis.yml
make deploy-crowdsec
```

Importante:
remover uma config em uso pode falhar enquanto houver serviços consumindo-a. Nesse caso, remova ou atualize o stack antes.

### 5. Prepare o arquivo `.env`

Use o exemplo como base:

```bash
cp crowdsec/.env.example crowdsec/.env
```

O conteúdo esperado é:

```env
CROWDSEC_BOUNCER_API_KEY=
```

Na primeira subida, a chave ainda não existe. Isso é normal neste desenho de stack, porque a chave é gerada pelo próprio `cscli` do CrowdSec depois que o serviço `crowdsec` já está rodando.

### 6. Faça o primeiro deploy do stack

Suba o CrowdSec:

```bash
make deploy-crowdsec
```

O `makefile` executa:

```bash
(set -a && source crowdsec/.env && set +a && docker stack deploy --detach=true -c crowdsec/docker-compose.yml crowdsec)
```

Neste primeiro deploy, o comportamento esperado pode ser:

- `crowdsec_crowdsec` sobe normalmente
- `crowdsec_bouncer-traefik` falha com erro de variável ausente

O erro típico do bouncer é:

```text
The required env var CROWDSEC_BOUNCER_API_KEY is not provided. Exiting
```

Isso não significa que o agente principal falhou. Significa apenas que a chave ainda não foi gerada.

### 7. Confirme que o serviço principal do CrowdSec está rodando

Confira os serviços:

```bash
docker stack services crowdsec
```

Confira as tasks:

```bash
docker service ps crowdsec_crowdsec
docker service ps crowdsec_bouncer-traefik
```

Se `crowdsec_crowdsec` estiver em `1/1`, já é possível gerar a chave do bouncer.

### 8. Descubra o nome do container do CrowdSec

Liste os containers do serviço:

```bash
docker ps --format '{{.Names}}' | rg '^crowdsec_crowdsec\.'
```

O resultado costuma ser algo como:

```text
crowdsec_crowdsec.1.xxxxxxxxxxxxx
```

### 9. Gere a chave do bouncer com `cscli`

Com o nome do container em mãos, execute:

```bash
docker exec <container-do-crowdsec> cscli bouncers add traefik-bouncer -o raw
```

Exemplo:

```bash
docker exec crowdsec_crowdsec.1.xxxxxxxxxxxxx cscli bouncers add traefik-bouncer -o raw
```

Esse comando retorna apenas a chave em formato bruto.

Observação:
se você executar o comando novamente com o mesmo nome e a versão do `cscli` não permitir duplicidade, remova o bouncer anterior ou use outro nome.

Para listar bouncers já cadastrados:

```bash
docker exec <container-do-crowdsec> cscli bouncers list
```

### 10. Grave a chave no `.env`

Preencha o arquivo [`crowdsec/.env`](/mnt/d/docker/crowdsec/.env) assim:

```env
CROWDSEC_BOUNCER_API_KEY=<chave-gerada-pelo-cscli>
```

Não comite a chave real no repositório.

### 11. Redeploye o stack

Depois de preencher o `.env`, suba novamente:

```bash
make deploy-crowdsec
```

Agora o esperado é:

- `crowdsec_crowdsec` em `1/1`
- `crowdsec_bouncer-traefik` em `1/1`

### 12. Valide o funcionamento final

Checagens principais:

```bash
docker stack services crowdsec
docker service ps crowdsec_crowdsec
docker service ps crowdsec_bouncer-traefik
docker service logs --tail 50 crowdsec_crowdsec
docker service logs --tail 50 crowdsec_bouncer-traefik
```

Quando estiver saudável, o bouncer deixa de reclamar da variável ausente e passa a expor o servidor HTTP na porta `8080`.

## Explicação Dos Componentes

### Serviço `crowdsec`

Definido em [`docker-compose.yml`](/mnt/d/docker/crowdsec/docker-compose.yml), ele:

- usa a imagem `crowdsecurity/crowdsec:latest`
- instala as coleções `crowdsecurity/linux` e `crowdsecurity/traefik`
- monta o banco local em `crowdsec-db`
- monta a configuração persistente em `crowdsec-config`
- monta os logs do Traefik em `/var/log/traefik` como somente leitura
- lê a config `crowdsec_acquis` em `/etc/crowdsec/acquis.yaml`

### Serviço `bouncer-traefik`

Também definido em [`docker-compose.yml`](/mnt/d/docker/crowdsec/docker-compose.yml), ele:

- usa a imagem `fbonalair/traefik-crowdsec-bouncer`
- depende da variável `CROWDSEC_BOUNCER_API_KEY`
- fala com o agente em `crowdsec_crowdsec:8080`
- publica o endpoint consultado pelo middleware `forwardAuth`

### Arquivo `config/acquis.yml`

O conteúdo atual é:

```yaml
filenames:
  - /var/log/traefik/*
labels:
  type: traefik
```

Isso diz ao CrowdSec para tratar esses arquivos como logs do Traefik.

## Como Ativar o Middleware Nos Serviços

Para proteger um router com CrowdSec, adicione o middleware `crowdsec@file` nas labels do serviço publicado pelo Traefik.

Exemplo:

```yaml
- "traefik.http.routers.app-secure.middlewares=crowdsec@file"
```

Se quiser combinar com outros middlewares:

```yaml
- "traefik.http.routers.app-secure.middlewares=crowdsec@file,authentik-headers"
```

Este repositório já tem exemplos desse padrão em serviços como `whoami` e `authentik`.

## Comandos Úteis

Deploy:

```bash
make deploy-crowdsec
```

Logs do agente:

```bash
docker service logs -f crowdsec_crowdsec
```

Logs do bouncer:

```bash
docker service logs -f crowdsec_bouncer-traefik
```

Forçar restart:

```bash
docker service update --force crowdsec_crowdsec
docker service update --force crowdsec_bouncer-traefik
```

Listar bouncers cadastrados:

```bash
docker exec <container-do-crowdsec> cscli bouncers list
```

## Troubleshooting

### Erro: `config not found: crowdsec_acquis`

Causa:
a config externa não foi criada no Swarm.

Correção:

```bash
docker config create crowdsec_acquis crowdsec/config/acquis.yml
```

### Erro: `CROWDSEC_BOUNCER_API_KEY is not provided`

Causa:
o arquivo `.env` está vazio ou sem a variável.

Correção:

1. suba o stack uma vez para colocar `crowdsec_crowdsec` em execução
2. gere a chave com `cscli bouncers add`
3. grave a chave em `crowdsec/.env`
4. faça novo deploy

### O bouncer continua reiniciando

Valide:

- se a chave no `.env` é exatamente a retornada pelo `cscli`
- se o deploy foi refeito após editar o `.env`
- se o serviço consegue resolver `crowdsec_crowdsec:8080`
- se os dois serviços estão na rede `traefik-public`

### O CrowdSec sobe, mas não analisa nada

Valide:

- se o Traefik está realmente gerando logs
- se o volume `traefik_traefik-logs` existe
- se `config/acquis.yml` aponta para o caminho correto
- se os arquivos aparecem dentro do container em `/var/log/traefik`

Exemplo de inspeção:

```bash
docker exec <container-do-crowdsec> ls -la /var/log/traefik
```

## Sequência Recomendada De Bootstrap

Se você estiver montando o ambiente do zero, a ordem mais segura é:

1. inicializar o Swarm
2. criar a rede `traefik-public`
3. subir o Traefik
4. criar a config `crowdsec_acquis`
5. copiar `.env.example` para `.env`
6. fazer o primeiro `make deploy-crowdsec`
7. gerar a chave do bouncer via `cscli`
8. preencher `crowdsec/.env`
9. rodar `make deploy-crowdsec` novamente
10. validar se os dois serviços ficaram em `1/1`

Esse é o bootstrap real exigido por esta stack.
