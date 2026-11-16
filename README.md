# Trabalho 4: Operacionalização segura de um serviço Linux com Docker

**Disciplina:** Sistemas Operacionais, Faculdade Serra Dourada (Análise e Desenvolvimento de Sistemas)
**Professor:** Guibson Krause
**Integrante:** Railan Alves Souza

## 1. Sobre o projeto

Serviço HTTP em Python (apenas biblioteca padrão) executado em container Docker. Ele responde na porta 8000 com o UID do processo, grava logs em stdout e tem uma rota `/crash` que simula uma falha controlada (saída com código 1), usada para demonstrar a recuperação automática.

## 2. Pré-requisitos

- Docker Desktop (com WSL2) em execução
- Git
- PowerShell (os comandos abaixo foram testados nele)

## 3. Estrutura do repositório

```
├── app.py            aplicação
├── Dockerfile        construção da imagem
├── compose.yaml      execução, restart e limites
├── .env.example      modelo de configuração
├── .gitignore        mantém o .env fora do Git
├── README.md         este documento (inclui o runbook)
└── evidencias/       saídas de terminal
```

## 4. Runbook

### 4.1 Configurar

```powershell
Copy-Item .env.example .env
```

| Variável | Função | Padrão |
|---|---|---|
| `PORT` | Porta em que a aplicação escuta | 8000 |
| `LOG_LEVEL` | Nível de log (INFO, DEBUG...) | INFO |
| `ENABLE_CRASH` | Habilita a rota `/crash` (somente em teste) | true |

Para mudar um valor sem editar o código: editar o `.env`, salvar e **recriar** o container (o `.env` é lido na criação).
O `.env` real fica fora do Git (`.gitignore`); só o `.env.example` é versionado.

### 4.2 Construir a imagem e iniciar

```powershell
docker compose up -d --build
```

### 4.3 Testar

```powershell
docker compose ps
curl.exe localhost:8000
```

Resposta esperada: `Servico no ar. UID do processo: 10001`
URL de teste no navegador: http://localhost:8000

### 4.4 Verificar usuário, estado, logs e limites

```powershell
docker exec servico-seguro id
docker ps --filter name=servico-seguro
docker ps -a --filter name=servico-seguro
docker compose logs
docker inspect -f "{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}" servico-seguro
docker inspect -f "{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.ReadonlyRootfs}}" servico-seguro
docker stats --no-stream servico-seguro
```

Resultados esperados:
- `id`: `uid=10001(appuser)`, ou seja, não é root.
- Memória `134217728` (128 MB) e CPU `500000000` (0,5 CPU).
- Política `on-failure` e sistema de arquivos somente leitura (`true`).

### 4.5 Simular a falha e confirmar a recuperação

```powershell
docker inspect -f "{{.RestartCount}}" servico-seguro     # antes: 0
curl.exe localhost:8000
Start-Sleep -Seconds 10                                  # container rodando pelo menos 10 s
curl.exe localhost:8000/crash                            # curl retorna "Empty reply": esperado
docker ps --filter name=servico-seguro
docker inspect -f "{{.RestartCount}}" servico-seguro     # depois: 1
docker compose logs
curl.exe localhost:8000                                  # o serviço voltou
```

### 4.6 Parar

```powershell
docker compose down
```

## 5. Decisões de segurança

| Medida | Configuração | Por quê |
|---|---|---|
| Usuário sem privilégios | `USER 10001` no Dockerfile | Se a aplicação for comprometida, o invasor não tem poderes de root |
| Porta alta | 8000 (> 1024) | Portas abaixo de 1024 exigem privilégio; a aplicação não precisa dele |
| Arquivos | Só leitura de `/app/app.py` | A aplicação não escreve em disco, o que permite `read_only: true` |
| Sem privilégios extras | `cap_drop: ALL` e `no-new-privileges` | Remove capacidades do kernel que o serviço não usa |
| Sem modo privilegiado | `privileged` não é usado | Exigência do enunciado |
| Limites de recursos | `mem_limit: 128m`, `cpus: 0.5` | Aplicados via cgroups (v2 no Docker Desktop/WSL2) |
| Credenciais | `.env` no `.gitignore` | Segredos reais não vão para o repositório |

## 6. Recuperação automática

`restart: on-failure` reinicia o container apenas quando o processo principal termina com código diferente de zero. A parada normal (`docker stop`, `docker compose down`) não dispara reinício.

## 7. Evidências

Saídas na pasta `evidencias/`:

| Arquivo | O que mostra |
|---|---|
| `01-compose-ps.txt` | Serviço no ar |
| `02-resposta-uid.txt` | Resposta HTTP com UID 10001 |
| `03-usuario.txt` | `id` dentro do container |
| `04-limites.txt` | Limites de memória e CPU aplicados |
| `05-stats.txt` | Consumo versus limite |
| `06-logs.txt` | Logs de inicialização e requisições |
| `07-falha-e-recuperacao.txt` | RestartCount antes/depois e logs da falha |

**Antes e depois da falha:** o `RestartCount` passou de 0 para 1; o log registrou `ERROR Falha controlada solicitada. Encerrando com codigo 1`, seguido de nova linha `Iniciando na porta 8000` cerca de 1 segundo depois; o `curl` final voltou a responder com UID 10001.