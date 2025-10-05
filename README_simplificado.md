# Nota de Orientação (adição sem alterar seus comandos)

> **Objetivo:** Entramos em um servidor **legado** com **pouca documentação**. A ideia é **se localizar rápido**,
> **achar a aplicação** e **investigar problemas**, **sem mexer** nos comandos já existentes no seu documento.

## Roteiro rápido (3 passos)
1. **Reconhecer o host:** `whoami`, `hostnamectl`, `cat /etc/os-release`, `df -h`, `ss -tulpn`.
2. **Localizar com `find`:** procure **serviços**, **configs** e **logs** nos diretórios comuns (`/etc`, `/opt`, `/srv`, `/var`, `/usr/local`).  
   - Exemplos (extras, sem alterar seu conteúdo original):  
     ```bash
     find /etc/systemd/system -type f -name "*.service" 2>/dev/null
     find / -xdev \( -name "*.env" -o -name "*.yml" -o -name "*.properties" -o -name "*.conf" \) -print 2>/dev/null
     find /var/log -type f -mtime -2 -size +0 -print 2>/dev/null
     ```
3. **Investigar e validar:** `systemctl status <serviço>`, `journalctl -u <serviço> -e`, `tail -f <log>`, `curl -I http://127.0.0.1:<porta>/health`.

### Dicas rápidas
- Redirecione erros para reduzir ruído: `2>/dev/null`.
- Registre **paths, portas, usuário de execução e comandos de restart** — vira uma mini-doc do legado.
- **Backup** antes de alterações e **comentários** explicando o que mudou.

---

# Material Complementar: Linux Básico para DevOps
## Investigação e Correção de Incidente na MariaLazaraCloud

---

### Informações do Curso
**Autora:** Maria Lazara (@marialazaradev)  
**Canal YouTube:** https://www.youtube.com/@marialazaradev  
**Metodologia:** Aprendizado Baseado em Problemas (PBL)  
**Empresa Fictícia:** MariaLazaraCloud (SaaS B2B)  
**Cenário:** Investigação e correção ativa de incidente em produção  
**Duração Estimada:** 2-3 horas  

---

## 📚 Índice

1. [Introdução ao Cenário](#introdução-ao-cenário)
2. [Objetivos de Aprendizado](#objetivos-de-aprendizado)
3. [Preparação do Ambiente](#preparação-do-ambiente)
4. [A História do Incidente](#a-história-do-incidente)
5. [Passo a Passo Detalhado](#passo-a-passo-detalhado)
6. [Conceitos Fundamentais](#conceitos-fundamentais)
7. [Glossário](#glossário)
8. [Referências e Materiais Adicionais](#referências-e-materiais-adicionais)

---

## 🎯 Introdução ao Cenário

### Contexto Empresarial
A **MariaLazaraCloud** é uma empresa de tecnologia que oferece soluções SaaS (Software as a Service) B2B para automação de cobrança e faturamento. 

### O Incidente
**Data/Hora:** 27 de setembro de 2024, 14:25 UTC  
**Serviço Afetado:** billing-api (API de processamento de pagamentos)  
**Sintomas Iniciais:**
- Aplicação não consegue processar transações
- Erros constantes de corrupção de dados nos logs
- Falha na validação de integridade dos arquivos
- Serviço rodando mas rejeitando todas as transações
- Dashboard mostrando status "degraded"

### Seu Papel
Você é o **DevOps Engineer on-call** responsável por:
- Investigar a causa raiz do problema
- Analisar logs e configurações do sistema
- Coordenar com o time para entender mudanças recentes
- Implementar a solução adequada (rollback)
- Validar a recuperação do serviço
- Documentar lições aprendidas

---

## 🎓 Objetivos de Aprendizado

Ao final desta aula, você será capaz de:

1. **Analisar processos** em execução no sistema Linux
2. **Monitorar logs** em tempo real para identificar problemas
3. **Navegar eficientemente** no sistema de arquivos Linux
4. **Investigar problemas** de integridade de dados
5. **Usar find** para localizar arquivos e diretórios
6. **Editar arquivos** de configuração usando nano
7. **Aplicar metodologia** completa de troubleshooting
8. **Executar rollback** baseado em documentação
9. **Validar recuperação** de serviços críticos

---

## 🔧 Preparação do Ambiente

### Executar o Container

```bash
# Baixar a imagem do laboratório
docker pull marialazaradev/linux-essentials:latest

# Executar o container
docker run -it --name devops-investigation marialazaradev/linux-essentials:latest
```

### Verificação Inicial
Ao conectar, você deve ver:
```
MariaLazaraCloud Billing API Server
Iniciando billing-api em background...
billing-api iniciado com PID 8
devops@prod-web-01:/srv/app$
```

---

## 📖 A História do Incidente

### Capítulo 1: O Alerta

Era uma tarde tranquila de sexta-feira quando seu telefone tocou. Como DevOps Engineer on-call da MariaLazaraCloud, você sabia que essa ligação poderia significar problemas.

"Temos um problema no billing-api", disse a voz do outro lado da linha. "Os clientes estão reportando que não conseguem processar pagamentos. O dashboard está mostrando status degraded há alguns minutos."

Você rapidamente abriu seu laptop e se conectou ao servidor de produção. Era hora de colocar suas habilidades de troubleshooting em ação.

### Capítulo 2: A Primeira Investigação

Conectado ao servidor `prod-web-01.marialazaracloud.com`, você começou sua investigação. O primeiro passo sempre é entender o que está acontecendo no sistema. Algo claramente estava errado - a aplicação billing-api estava rodando, mas não funcionando corretamente.

Os logs mostravam erros constantes, algo sobre "data corruption detected" e "data validation failed". Sua experiência te disse que isso poderia ser um problema de integridade de dados.

### Capítulo 3: A Comunicação com o Time

Após identificar os primeiros sintomas, você imediatamente comunicou suas descobertas ao time através do canal #incidents no Slack:

**DevOps Engineer (você):** "🚨 INCIDENTE: billing-api com problemas de integridade de dados. Investigando... Status: INVESTIGATING"

**Tech Lead:** "Oi! Ontem tivemos uma GMUD executada. CHG-2024-0927-001 - otimização de performance de I/O. Pode estar relacionado?"

**DevOps Engineer (você):** "Perfeito! Vou verificar a documentação da GMUD. Pode me passar os detalhes?"

**Tech Lead:** "Documentação está em /opt/docs/GMUD-CHG-2024-0927-001.txt no servidor. João Silva executou ontem às 22:30 UTC."

### Capítulo 4: A Descoberta

Com essa informação crucial, você agora tinha uma pista valiosa. Mudanças recentes são sempre suspeitas quando problemas aparecem. Era hora de investigar a documentação da GMUD e entender exatamente o que foi alterado na migração de dados.

A partir daqui, você seguirá um processo sistemático de investigação, análise da documentação de mudança, e implementação da solução adequada.

---

## 🔍 Passo a Passo Detalhado

### Passo 1: Reconhecimento do Sistema

#### Comando
```bash
uname -a
```

#### Retorno Esperado
```
Linux prod-web-01 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64 GNU/Linux
```

#### Explicação Detalhada
O comando `uname -a` exibe informações completas do sistema operacional, confirmando que estamos no ambiente correto para investigação.

#### Conceitos-Chave
- **System Information**: Identificação do sistema operacional
- **Environment Verification**: Verificação do ambiente de trabalho

---

### Passo 2: Verificação de Processos em Execução

#### Comando
```bash
ps aux
```

#### Retorno Esperado
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
devops       1  0.0  0.1   4624  1024 pts/0    Ss   14:20   0:00 /bin/bash -c /usr/local/bin/start-billing-api.sh && /bin/bash
devops       8  0.1  0.5  15832  5120 pts/0    S    14:20   0:00 python3 /usr/local/bin/billing-api-simulator.py
devops      15  0.0  0.1   7236  1280 pts/0    S+   14:21   0:00 ps aux
```

#### Comando Específico para Billing API
```bash
ps aux | grep billing
```

#### Retorno Esperado
```
devops       8  0.1  0.5  15832  5120 pts/0    S    14:20   0:00 python3 /usr/local/bin/billing-api-simulator.py
devops      16  0.0  0.0   6208   896 pts/0    S+   14:21   0:00 grep billing
```

#### Explicação Detalhada

**Análise dos processos:**
- **PID 8**: billing-api está rodando
- **Estado S**: Processo está em sleep (aguardando)
- **Aplicação ativa**: Processo em execução mas com problemas

#### Conceitos-Chave
- **Process Management**: Gerenciamento de processos no Linux
- **Process States**: Estados dos processos do sistema

---

### Passo 3: Análise Inicial dos Logs

#### Comando
```bash
cd /var/log/marialazaracloud/billing
tail app.log
```

#### Retorno Esperado
```
2024-09-27 14:20:18 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:20:18 ERROR Cannot process transactions - data validation failed
2024-09-27 14:20:23 INFO Using data directory: /data/billing
2024-09-27 14:20:24 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:20:24 ERROR Cannot process transactions - data validation failed
2024-09-27 14:20:29 INFO Using data directory: /data/billing
2024-09-27 14:20:30 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:20:30 ERROR Cannot process transactions - data validation failed
```

#### Explicação Detalhada

**Primeira descoberta importante:**
- Aplicação está tentando acessar `/data/billing`
- Erro "Data corruption detected" constante  
- Problema parece ser de integridade dos dados
- **Suspeita**: Algo pode ter sido corrompido na migração

#### Conceitos-Chave
- **Log Analysis**: Análise de logs para identificar problemas
- **Pattern Recognition**: Reconhecimento de padrões de erro

---

### Passo 4: Monitoramento em Tempo Real

#### Comando
```bash
tail -f app.log
```

#### Retorno Esperado (Streaming)
```
2024-09-27 14:35:18 INFO Using data directory: /data/billing
2024-09-27 14:35:19 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:35:19 ERROR Cannot process transactions - data validation failed
2024-09-27 14:35:24 INFO Using data directory: /data/billing
2024-09-27 14:35:25 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:35:25 ERROR Cannot process transactions - data validation failed
(continua em tempo real...)
```

#### Como Parar
```
Ctrl+C (para sair do tail -f)
```

#### Explicação Detalhada

**Observação crucial:**
- Problema é ativo e contínuo
- Erro se repete a cada ~5 segundos
- Aplicação está tentando usar `/data/billing` mas detectando corrupção

---

### Passo 5: Investigação da Documentação da GMUD

Agora que o time informou sobre a GMUD executada ontem, é hora de investigar o que foi alterado.

#### Comando
```bash
cd /opt/docs
cat GMUD-CHG-2024-0927-001.txt
```

#### Retorno Esperado
```
=== GESTÃO DE MUDANÇA - GMUD ===
Ticket: CHG-2024-0927-001
Data: 26/09/2024 22:30 UTC
Responsável: João Silva (DevOps)
Aprovador: Maria Santos (Tech Lead)

OBJETIVO:
Otimização de performance de I/O do billing-api
Migrar dados de transação para storage de alta performance

MUDANÇAS PLANEJADAS:
1. Criar novo diretório de dados em /data/billing
2. Migrar dados de /opt/seeds para /data/billing  
3. Atualizar configuração data_directory em billing-config.yml
4. Configurar permissões apropriadas no novo diretório

ROLLBACK PLAN:
Se houver problemas:
1. Reverter data_directory para: /opt/seeds
2. Reiniciar billing-api service
3. Validar funcionamento pelos logs

JANELA DE MANUTENÇÃO: 22:30-23:00 UTC
STATUS: EXECUTADO ✓
```

#### Explicação Detalhada

**Descoberta crucial:**
- GMUD foi executada ontem durante janela de manutenção
- Objetivo era migrar dados de `/opt/seeds` para `/data/billing`
- Configuração deveria ser alterada para apontar para novo diretório
- **Há um rollback plan documentado!**

---

### Passo 6: Análise do Log de Execução da GMUD

#### Comando
```bash
cat change-log.txt
```

#### Retorno Esperado
```
=== LOG DE MUDANÇAS DO SISTEMA ===

26/09/2024 22:35 UTC - CHG-2024-0927-001
[João Silva] Criando novo diretório de dados
$ mkdir -p /data/billing
$ chown devops:devops /data/billing
$ chmod 755 /data/billing

26/09/2024 22:37 UTC - CHG-2024-0927-001  
[João Silva] Copiando dados para novo local
$ cp /opt/seeds/transactions /data/billing/
$ cp /opt/seeds/customers.txt /data/billing/

26/09/2024 22:40 UTC - CHG-2024-0927-001
[João Silva] Atualizando configuração
ANTES: data_directory: "/opt/seeds"
DEPOIS: data_directory: "/data/billing"

26/09/2024 22:42 UTC - CHG-2024-0927-001
[João Silva] Reiniciando serviço
$ systemctl restart billing-api

26/09/2024 22:45 UTC - CHG-2024-0927-001
[João Silva] Validação básica passou, GMUD concluída com sucesso
```

#### Explicação Detalhada

**Procedimento da GMUD parece correto:**
- Diretório foi criado com permissões adequadas
- Dados foram copiados de `/opt/seeds` para `/data/billing`
- Configuração foi atualizada corretamente
- **Mas algo deu errado na integridade dos dados**

---

### Passo 7: Verificação da Configuração Atual

#### Comando
```bash
cd /etc/marialazaracloud
cat billing-config.yml
```

#### Retorno Esperado
```yaml
database:
  host: "db.marialazaracloud.internal"
  port: 5432
  name: "billing_prod"
data_directory: "/data/billing"
logging:
  level: "INFO"
  path: "/var/log/marialazaracloud/billing"
processing:
  batch_size: 100
  timeout: 30
```

#### Explicação Detalhada

**Configuração está correta:**
- Configuração atual aponta para `/data/billing`
- Path está correto, problema deve ser nos dados mesmo

---

### Passo 8: Investigação do Diretório de Dados

#### Comando
```bash
ls -la /data/
```

#### Retorno Esperado
```
total 12
drwxr-xr-x  3 root root 4096 Sep 27 14:35 .
drwxr-xr-x 17 root root 4096 Sep 27 14:35 ..
drwxr-xr-x  2 devops devops 4096 Sep 27 14:35 billing
```

#### Comando para Verificar Permissões
```bash
ls -la /data/billing/
```

#### Retorno Esperado
```
total 16
drwxr-xr-x 2 devops devops 4096 Sep 27 14:35 .
drwxr-xr-x 3 root   root   4096 Sep 27 14:35 ..
-rw-r--r-- 1 devops devops   45 Sep 27 14:35 customers.txt
-rw-r--r-- 1 devops devops   70 Sep 27 14:35 transactions
```

#### Explicação Detalhada

**Permissões estão corretas:**
- `/data/billing` existe e tem permissões de devops
- Usuário `devops` consegue acessar os arquivos
- **Problema deve estar no conteúdo dos arquivos**

---

### Passo 9: Verificação da Integridade dos Dados

#### Comando para Verificar Arquivo de Transações
```bash
cat /data/billing/transactions
```

#### Retorno Esperado
```
ERROR_MIGRATION_INCOMPLETE_DATA_CORRUPTED_2024
BACKUP_REQUIRED_ROLLBACK_RECOMMENDED
```

#### Comando para Verificar Customers
```bash
cat /data/billing/customers.txt
```

#### Retorno Esperado
```
MIGRATION_FAILED_DATA_INTEGRITY_COMPROMISED
```

#### Explicação Detalhada

**🎯 CAUSA RAIZ ENCONTRADA!**
- Os dados em `/data/billing` foram corrompidos durante a migração
- Arquivos contêm mensagens de erro em vez dos dados reais
- A aplicação detecta a corrupção e rejeita o processamento
- **Necessário fazer rollback para os dados originais**

---

### Passo 10: Verificação dos Dados Originais

#### Comando
```bash
ls -la /opt/seeds/
```

#### Retorno Esperado
```
total 20
drwxr-xr-x 2 devops devops 4096 Sep 27 14:35 .
drwxr-xr-x 3 devops devops 4096 Sep 27 14:35 ..
-rw-r--r-- 1 devops devops  156 Sep 27 14:35 customers.txt
-rw------- 1 devops devops   87 Sep 27 14:35 public_data.txt
-rw-r--r-- 1 devops devops  165 Sep 27 14:35 transactions
```

#### Comando para Verificar Dados Originais
```bash
cat /opt/seeds/transactions
```

#### Retorno Esperado
```
TXN001,2024-09-27,cust_123,29.99,pending
TXN002,2024-09-27,cust_456,99.99,pending
TXN003,2024-09-27,cust_789,49.99,pending
TXN004,2024-09-27,cust_321,199.99,pending
TXN005,2024-09-27,cust_654,79.99,pending
```

#### Explicação Detalhada

**Dados originais estão íntegros:**
- `/opt/seeds` contém os dados corretos
- Arquivo `transactions` tem as transações válidas
- **Decisão: fazer rollback da configuração para /opt/seeds**

---

### Passo 11: Comparação com Backup da Configuração

#### Comando
```bash
diff billing-config.yml billing-config.yml.backup
```

#### Retorno Esperado
```
4c4
< data_directory: "/data/billing"
---
> data_directory: "/opt/seeds"
```

#### Explicação Detalhada

**Decisão de rollback confirmada:**
- Configuração original apontava para `/opt/seeds`
- Dados em `/opt/seeds` estão íntegros
- Seguindo o rollback plan da GMUD: reverter para estado anterior

---

### Passo 12: Preparação para Edição Segura da Configuração

#### Comando para Criar Backup Adicional
```bash
cp billing-config.yml billing-config.yml.pre-rollback
```

#### Comando para Verificar Permissões Atuais
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-r--r--r-- 1 devops devops 287 Sep 27 14:35 billing-config.yml
```

#### Comando para Permitir Edição
```bash
chmod 644 billing-config.yml
```

#### Verificação Pós-Alteração
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-rw-r--r-- 1 devops devops 287 Sep 27 14:35 billing-config.yml
```

#### Explicação Detalhada

**Preparação para edição segura:**
- Backup adicional criado antes de qualquer alteração
- Arquivo estava com permissões somente leitura (444)
- Alterado para 644 (proprietário pode escrever)
- Boa prática: sempre fazer backup antes de editar configs críticas

#### Conceitos-Chave
- **Configuration Backup**: Backup preventivo antes de alterações
- **Permission Management**: Gerenciamento seguro de permissões
- **Safe Editing**: Edição segura de arquivos críticos

---

### Passo 13: Execução do Rollback Conforme Documentado

#### Comando
```bash
nano billing-config.yml
```

#### Ação Necessária
1. Alterar `/data/billing` para `/opt/seeds`
2. Salvar com `Ctrl+O`, Enter
3. Sair com `Ctrl+X`

#### Resultado Esperado
```yaml
database:
  host: "db.marialazaracloud.internal"
  port: 5432
  name: "billing_prod"
data_directory: "/opt/seeds"
logging:
  level: "INFO"
  path: "/var/log/marialazaracloud/billing"
processing:
  batch_size: 100
  timeout: 30
```

---

### Passo 14: Restauração de Permissões Seguras

#### Comando para Restaurar Permissões Restritivas
```bash
chmod 444 billing-config.yml
```

#### Verificação da Segurança
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-r--r--r-- 1 devops devops 287 Sep 27 14:40 billing-config.yml
```

#### Explicação Detalhada

**Segurança restaurada:**
- Arquivo retornado para permissões somente leitura (444)
- Impede alterações acidentais da configuração crítica
- Boa prática de segurança: configurações devem ser protegidas contra escrita

#### Conceitos-Chave
- **Security Hardening**: Endurecimento de segurança após edição
- **Read-Only Protection**: Proteção contra alterações acidentais
- **Configuration Security**: Segurança de arquivos de configuração

---

### Passo 15: Reinicialização da Aplicação

#### Comando para Parar a Aplicação
```bash
ps aux | grep billing
kill 8
```

#### Comando para Reiniciar
```bash
/usr/local/bin/start-billing-api.sh
```

#### Retorno Esperado
```
Iniciando billing-api em background...
billing-api iniciado com PID 48
```

---

### Passo 16: Validação da Recuperação

#### Comando
```bash
cd /var/log/marialazaracloud/billing
tail -f app.log
```

#### Retorno Esperado Após Rollback
```
2024-09-27 14:41:15 INFO Starting billing-api service
2024-09-27 14:41:16 INFO Initializing database connections
2024-09-27 14:41:17 INFO Using data directory: /opt/seeds
2024-09-27 14:41:18 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:18 INFO Transaction validation completed successfully
2024-09-27 14:41:22 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:22 INFO Transaction validation completed successfully
2024-09-27 14:41:26 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:26 INFO Transaction validation completed successfully
```

#### Explicação Detalhada

**🎉 RECUPERAÇÃO COMPLETA!**
- Sem erros de "Data corruption detected"
- Logs mostram processamento normal
- Path correto `/opt/seeds` sendo usado
- Transações sendo processadas com sucesso
- **Rollback executado com sucesso!**

---

### Passo 17: Comunicação da Resolução

Agora é hora de informar ao time sobre a resolução:

**DevOps Engineer (você):** "✅ RESOLUÇÃO: Rollback executado com sucesso conforme GMUD CHG-2024-0927-001. Causa raiz: corrupção de dados durante migração para /data/billing - arquivos de transações foram corrompidos. Dados originais em /opt/seeds estão íntegros. Sistema operacional normalmente."

**Tech Lead:** "Perfeito! Obrigada pela investigação rápida. Vamos revisar o processo de validação de integridade de dados nas GMUDs."

---

## 📊 Métricas de Recuperação

#### Comando para Contar Erros de Corrupção Antes do Rollback
```bash
grep "Data corruption detected" app.log | wc -l
```

#### Retorno Esperado
```
15
```

#### Comando para Contar Sucessos Pós-Rollback
```bash
grep "Transaction validation completed successfully" app.log | wc -l
```

#### Retorno Esperado
```
8
```

#### Comando para Ver Últimas 10 Linhas
```bash
tail -n 10 app.log
```

#### Retorno Esperado
```
2024-09-27 14:41:34 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:34 INFO Transaction validation completed successfully
2024-09-27 14:41:38 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:38 INFO Transaction validation completed successfully
2024-09-27 14:41:42 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:42 INFO Transaction validation completed successfully
2024-09-27 14:41:46 INFO Processing transaction batch from /opt/seeds
2024-09-27 14:41:46 INFO Transaction validation completed successfully
2024-09-27 14:41:50 WARN Retrying failed transaction validation
2024-09-27 14:41:54 INFO Processing transaction batch from /opt/seeds
```

#### Explicação Detalhada

**Confirmação quantitativa do rollback:**
- 15 erros de "Data corruption detected" antes do rollback
- 8+ sucessos após rollback
- Últimas 10 linhas mostram operação normal
- **GMUD rollback executado com sucesso**

**Lições aprendidas:**
- Importância de verificar integridade de dados durante GMUD
- Documentação facilitou identificação rápida da causa
- Rollback plan permitiu recuperação rápida
- Validação de integridade é essencial em migrações
- Change Management formal é essencial para rastreabilidade

---

## 🛠️ Conceitos Fundamentais

### 1. Sistema de Permissões Unix

#### Tipos de Permissão
- **r (read)**: Leitura de arquivo ou listagem de diretório
- **w (write)**: Escrita em arquivo ou modificação de diretório  
- **x (execute)**: Execução de arquivo ou acesso a diretório

#### Níveis de Acesso
- **Owner (u)**: Proprietário do arquivo
- **Group (g)**: Grupo proprietário
- **Others (o)**: Todos os outros usuários

#### Notação Octal
- **644**: rw-r--r-- (owner read/write, others read)
- **755**: rwxr-xr-x (owner full, others read/execute)

### 2. Gerenciamento de Processos

#### Comandos Essenciais
- `ps aux`: Listar todos os processos
- `kill PID`: Terminar processo graciosamente
- `kill -9 PID`: Terminar processo forçadamente

### 3. Análise de Logs

#### Estratégias de Monitoramento
- **Real-time**: `tail -f` para monitoramento contínuo
- **Historical**: `head`/`tail` para análise temporal
- **Pattern matching**: `grep` para filtrar eventos

### 4. Integridade de Dados

#### Validação de Dados
- **Checksum verification**: Verificação de integridade através de hash
- **Content validation**: Validação do formato e conteúdo dos dados
- **Backup verification**: Verificação de integridade dos backups

### 5. Change Management (GMUD)

#### Elementos Essenciais
- **Documentação**: Registro detalhado das mudanças
- **Rollback Plan**: Procedimento de reversão documentado
- **Validação**: Verificação pós-implementação
- **Comunicação**: Informação ao time sobre mudanças

---

## 📝 Glossário

**GMUD**: Gestão de Mudança - processo formal para implementar alterações em produção

**Rollback**: Reversão de uma mudança para o estado anterior

**Root Cause**: Causa fundamental de um problema

**PID**: Process ID - identificador único de processo

**Log Analysis**: Análise de arquivos de log para identificação de problemas

**Data Corruption**: Corrupção de dados - alteração não intencional de dados

**Data Integrity**: Integridade de dados - precisão e consistência dos dados

**Configuration File**: Arquivo que contém parâmetros de configuração

**Troubleshooting**: Processo sistemático de identificar e resolver problemas

---

## 🎯 Lições Aprendidas

### Do Incidente
1. **Validação de integridade de dados** durante migrações é crítica
2. **Documentação detalhada** facilita troubleshooting rápido  
3. **Rollback plans** permitem recuperação eficiente
4. **Falhas silenciosas** são perigosas e difíceis de detectar
5. **Change Management** formal é essencial para rastreabilidade

### Para Prevenção Futura
1. Implementar checksums para validação de integridade
2. Testes de validação de dados pós-migração
3. Verificação automática de conteúdo de arquivos críticos
4. Backup completo antes de qualquer migração
5. Validação em múltiplas camadas (filesystem + aplicação)

---

## 📚 Referências e Materiais Adicionais

### Comandos de Execução

#### Executar o Laboratório
```bash
docker pull marialazaradev/linux-essentials:latest
docker run -it --name devops-investigation marialazaradev/linux-essentials:latest
```

### Próximos Passos
1. Praticar troubleshooting de integridade de dados em ambiente real
2. Estudar ferramentas avançadas de monitoramento
3. Aprender shell scripting para automação de diagnósticos
4. Explorar ferramentas de log management
5. Aprofundar em segurança de filesystem

---

## 🎬 Conclusão

Neste laboratório, você vivenciou um cenário real de troubleshooting em ambiente DevOps, desde a identificação inicial do problema até a implementação bem-sucedida do rollback. 

Você aprendeu a:
- Investigar problemas de forma sistemática
- Analisar documentação de mudanças (GMUD)
- Identificar causa raiz através de análise de logs
- Executar rollback seguindo procedimentos documentados
- Validar a recuperação completa do serviço

A experiência mostrou como a documentação adequada e processos bem estruturados de change management são fundamentais para a recuperação rápida de incidentes em produção.

---

## 👋 Agradecimentos

Este material complementar foi criado pela **Maria Lazara** como apoio ao tutorial completo disponível no YouTube. 

Se este conteúdo foi útil para você:

📺 **Inscreva-se no canal**: https://www.youtube.com/@marialazaradev
👍 **Deixe seu like** no vídeo correspondente
💬 **Comente** suas dúvidas e experiências
📤 **Compartilhe** com outros profissionais da área

Seu feedback é muito importante para continuar criando conteúdo de qualidade para a comunidade DevOps!

### Conecte-se Comigo
- **YouTube**: @marialazaradev
- **Conteúdo**: Tutoriais práticos de DevOps, Linux e Cloud

Obrigada por estudar comigo! 🚀

---

**Fim do Material Complementar**

*Este documento foi criado para fins educacionais, focando em resolução prática de problemas comuns no ambiente DevOps.*