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
A **MariaLazaraCloud** é uma empresa de tecnologia que oferece soluções SaaS B2B para automação de cobrança e faturamento. A arquitetura usa microserviços em containers Docker, com foco em disponibilidade.

### O Incidente
**Data/Hora:** 27 de setembro de 2024, 14:25 UTC  
**Serviço Afetado:** billing-api (API de pagamentos)  
**Sintomas Iniciais:**
- Aplicação não processa transações
- Erros de corrupção de dados nos logs
- Falha na validação de integridade
- Serviço rodando, mas rejeitando transações
- Dashboard em "degraded"

### Seu Papel
Você é o DevOps Engineer on-call. Como o sistema é legado e tem pouca documentação, você não conhece bem os caminhos dos arquivos. Vamos explorar o sistema calmamente, usando comandos básicos como `find` apenas quando necessário para localizar arquivos e diretórios. Cada passo inclui explicações detalhadas dos comandos (o que cada parte faz), e reflexões sobre por quê usá-los, quando aplicá-los, o motivo e o objetivo. Assumimos que você é iniciante no Linux, então vamos devagar, com narrativas explicando o que estamos fazendo antes de prosseguir para o próximo passo.

---

## 🎓 Objetivos de Aprendizado

Ao final, você será capaz de:

1. Analisar processos no Linux
2. Monitorar logs em tempo real
3. Navegar no sistema de arquivos
4. Investigar integridade de dados
5. Usar `find` para localizar arquivos
6. Editar arquivos com `nano`
7. Aplicar troubleshooting sistemático
8. Executar rollback
9. Validar recuperação

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
Ao conectar, você verá:
```
MariaLazaraCloud Billing API Server
Iniciando billing-api em background...
billing-api iniciado com PID 8
devops@prod-web-01:/srv/app$
```

---

## 📖 A História do Incidente

Era uma tarde de sexta quando você recebeu um alerta: o billing-api não processa pagamentos. Como DevOps on-call, você se conecta ao servidor legado `prod-web-01`. Sem muita documentação, você começa explorando o sistema com comandos simples para entender o ambiente, analisar logs e configurações. Os logs mostram erros de corrupção de dados. O time menciona uma mudança recente. Após investigar, descobre corrupção nos dados migrados e faz um rollback simples.

---

## 🔍 Passo a Passo Detalhado

Vamos devagar, como se estivéssemos investigando juntos. Em cada fase, antes de partir para a próxima, eu explico o que estamos fazendo, o que encontramos até agora, por que isso importa e qual é o próximo passo lógico. Por exemplo, se encontrarmos caminhos relevantes, discutimos qual priorizar e por quê. Isso ajuda a entender o raciocínio por trás do troubleshooting.

### Passo 1: Reconhecimento do Sistema

Beleza, estamos conectados ao servidor, mas como é um sistema legado, não sabemos muito sobre ele. Vamos começar pelo básico: verificar as informações do sistema para confirmar que estamos no ambiente certo e entender sua versão. Isso pode indicar se há limitações ou compatibilidades que afetem nossa investigação. Se for uma versão antiga, por exemplo, alguns comandos modernos podem não funcionar.

#### Comando
```bash
uname -a
```

#### Retorno Esperado
```
Linux prod-web-01 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64 GNU/Linux
```

#### Explicação Detalhada do Comando
- **uname**: Comando base para mostrar informações sobre o sistema operacional (kernel, versão, arquitetura).
- **-a**: Flag (opção) que significa "all" (todos), exibindo detalhes completos como nome do host, versão do kernel, data de compilação e arquitetura (ex: x86_64 para 64 bits).

#### Reflexões
- **Por quê usar aqui?** Objetivo: Confirmar o ambiente (versão do Linux, arquitetura) para garantir que estamos no servidor correto e entender se há limitações (ex: versão antiga pode afetar comandos). Motivo: Em troubleshooting, sempre comece verificando o básico para contextualizar problemas.
- **Quando usar?** No início de qualquer sessão em um servidor desconhecido, ou quando suspeita de incompatibilidades de software/hardware.
- **Motivo e objetivo geral:** Evita erros ao assumir o ambiente errado; promove compreensão rápida do sistema.

Então, vimos que é um Linux Debian recente (kernel 6.1), arquitetura 64 bits. Isso é bom, significa que comandos padrão devem funcionar sem problemas. Agora, como o incidente é no billing-api, vamos checar se o serviço está rodando – isso pode indicar se o problema é no processo em si ou em algo que ele depende, como dados ou configs.

### Passo 2: Verificação de Processos em Execução

Ok, confirmamos o sistema. Agora, vamos ver os processos rodando, porque o sintoma é que o serviço está "degraded" mas rodando. Isso pode ser um app travado consumindo recursos ou aguardando algo. Se não encontrar o processo, pode ser que ele tenha crashed; se encontrar, podemos notar alto uso de CPU ou memória, o que aponta para loops infinitos ou leaks.

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

#### Explicação Detalhada do Comando
- **ps**: Comando base para "process status" (status de processos), lista processos rodando no sistema.
- **a**: Flag para mostrar processos de todos os usuários (não só o seu).
- **u**: Flag para exibir detalhes amigáveis ao usuário (como %CPU, %MEM, usuário dono).
- **x**: Flag para incluir processos sem terminal associado (ex: daemons em background).

#### Comando Específico para Billing API
```bash
ps aux | grep billing
```

#### Retorno Esperado
```
devops       8  0.1  0.5  15832  5120 pts/0    S    14:20   0:00 python3 /usr/local/bin/billing-api-simulator.py
devops      16  0.0  0.0   6208   896 pts/0    S+   14:21   0:00 grep billing
```

#### Explicação Detalhada do Comando
- **ps aux**: Como acima.
- **|**: Pipe (tubo), redireciona a saída de um comando para o próximo.
- **grep**: Comando base para buscar padrões em texto (global regular expression print).
- **billing**: Argumento, a palavra a buscar (filtra linhas com "billing").

#### Reflexões
- **Por quê usar aqui?** Objetivo: Ver se o billing-api está rodando e seu status (ex: PID 8, estado S para sleep). Motivo: Sintomas indicam serviço ativo mas falhando, então confirmamos processos para identificar candidatos a problemas.
- **Quando usar?** Quando suspeita de apps travados, alto uso de CPU/memória, ou para matar processos problemáticos.
- **Motivo e objetivo geral:** Processos são o coração do sistema; monitorá-los ajuda a diagnosticar lentidão ou falhas, promovendo gerenciamento eficiente de recursos.

Então, já vimos que o billing-api está rodando (PID 8, em Python), mas em estado "S" (sleep, aguardando algo). Uso de recursos baixo, então não é loop infinito. Isso sugere que o problema não é no processo em si, mas talvez em dados ou configs que ele usa. Próximo: vamos olhar os logs, pois eles registram erros internos. Como é legado, usamos `find` para achar onde estão os logs relacionados a "billing".

### Passo 3: Análise Inicial dos Logs

Beleza, o processo está vivo, mas pode estar logando erros. Vamos usar `find` para localizar diretórios chamados "billing", porque logs geralmente ficam em /var/log (um diretório padrão no Linux para logs de sistema e apps, organizado por app para evitar bagunça). Se encontrar múltiplos, priorizamos o de logs primeiro, pois erros aparecem lá. Isso pode revelar a causa raiz, como "corrupção de dados".

#### Comando para Encontrar Logs (se necessário)
```bash
find / -type d -name "billing" 2>/dev/null
```

#### Retorno Esperado (exemplo)
```
/var/log/marialazaracloud/billing
/data/billing
```

#### Explicação Detalhada do Comando
- **find**: Comando base para buscar arquivos/diretórios no filesystem.
- **/**: Argumento, raiz do sistema (busca em todo o disco a partir da raiz).
- **-type d**: Flag -type especifica tipo; d para diretórios (não arquivos).
- **-name "billing"**: Flag -name busca pelo nome exato (aqui, diretórios chamados "billing").
- **2>/dev/null**: Redireciona erros (permissões negadas) para /dev/null (lixo), evitando poluição na saída.

Beleza, encontramos dois paths com "billing": um em /var/log/... (isso é logs, onde apps escrevem mensagens de erro/info) e outro em /data/... (provavelmente dados). Vamos priorizar o de logs primeiro, porque queremos ver os erros reportados. Navegamos para lá e olhamos as últimas linhas do app.log com `tail`, para focar no recente.

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

#### Explicação Detalhada do Comando
- **cd**: Comando base para "change directory" (mudar diretório atual).
- **/var/log/marialazaracloud/billing**: Argumento, caminho completo para o diretório de logs.
- **tail**: Comando base para mostrar o final de um arquivo (útil para logs recentes).
- **app.log**: Argumento, o arquivo específico a ler (sem flags, padrão é últimas 10 linhas).

#### Reflexões
- **Por quê usar aqui?** Objetivo: Identificar erros recentes nos logs para pinpointar o problema (ex: corrupção em /data/billing). Motivo: Logs registram eventos; são a "caixa preta" do app.
- **Quando usar?** Em qualquer falha de app, para ver mensagens de erro/tempo real.
- **Motivo e objetivo geral:** Logs ajudam a rastrear causas sem adivinhação; promovem diagnóstico baseado em evidências, economizando tempo.

Então, já vimos que tem erro constante de "Data corruption detected" em /data/billing/transactions. Isso pode ser corrupção de arquivos durante uma migração ou cópia falha – comum em legados sem checksums. O app usa /data/billing, mas falha na validação. Para confirmar se é contínuo, vamos monitorar em tempo real com `tail -f`, para ver se os erros persistem agora.

### Passo 4: Monitoramento em Tempo Real

Ok, os logs mostram erros repetidos. Vamos monitorar ao vivo para ver se continua acontecendo – isso confirma que o problema é ativo, não histórico, e pode mostrar padrões como intervalo de 5 segundos, indicando tentativas periódicas falhando.

#### Comando
```bash
tail -f app.log
```

#### Retorno Esperado (Streaming)
```
2024-09-27 14:35:18 INFO Using data directory: /data/billing
2024-09-27 14:35:19 ERROR Data corruption detected in /data/billing/transactions
2024-09-27 14:35:19 ERROR Cannot process transactions - data validation failed
(continua...)
```

#### Como Parar
Pressione Ctrl+C.

#### Explicação Detalhada do Comando
- **tail**: Como acima.
- **-f**: Flag para "follow" (seguir), monitora o arquivo em tempo real à medida que novas linhas são adicionadas.
- **app.log**: Argumento, o arquivo a monitorar.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Observar erros acontecendo ao vivo para confirmar se é contínuo. Motivo: Ajuda a ver padrões repetitivos.
- **Quando usar?** Para monitorar apps em execução, como servidores web ou serviços.
- **Motivo e objetivo geral:** Permite reação imediata a eventos; útil em produção para detectar falhas em andamento.

Então, confirmamos: erros a cada ~5 segundos, usando /data/billing. Isso reforça suspeita de corrupção nos dados. O time mencionou uma GMUD recente – mudanças são suspeitas comuns. Vamos achar a doc da GMUD com `find`, pois em legados, docs podem estar espalhadas. Buscamos por "GMUD-*", que é o padrão de nome.

### Passo 5: Investigação da Documentação da GMUD

Beleza, logs apontam para corrupção pós-migração provável. Vamos localizar a doc da GMUD para entender o que mudou – isso pode revelar se houve migração de dados e um plano de rollback, economizando tempo.

#### Comando para Encontrar Documentação (se necessário)
```bash
find / -name "GMUD-*" 2>/dev/null
```

#### Retorno Esperado (exemplo)
```
/opt/docs/GMUD-CHG-2024-0927-001.txt
```

#### Explicação Detalhada do Comando
- **find /**: Como acima, busca da raiz.
- **-name "GMUD-*"**: -name busca nome; "GMUD-*" usa wildcard * para qualquer coisa após "GMUD-" (ex: arquivos como GMUD-CHG-...).
- **2>/dev/null**: Ignora erros.

Agora, navegamos para lá e lemos com `cat`.

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

#### Explicação Detalhada do Comando
- **cd /opt/docs**: Muda para o diretório de docs.
- **cat**: Comando base para "concatenate" (concatenar), mas usado para exibir conteúdo de arquivos texto.
- **GMUD-CHG-2024-0927-001.txt**: Argumento, o arquivo a exibir.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Entender mudanças recentes que podem causar o incidente. Motivo: Mudanças são comuns culpadas em falhas.
- **Quando usar?** Após alertas pós-manutenação, para revisar docs.
- **Motivo e objetivo geral:** Documentação fornece contexto histórico; ajuda a evitar repetição de erros e planejar reversões.

Então, encontramos: GMUD migrando de /opt/seeds para /data/billing, com rollback plan! Isso explica os logs – corrupção na migração. Próximo: checar o log de execução da GMUD para ver se a cópia falhou de algum modo visível.

### Passo 6: Análise do Log de Execução da GMUD

Ok, a doc menciona mudanças, mas vamos ver o log detalhado para confirmar os passos executados – pode mostrar se a cópia foi incompleta ou com erros não reportados.

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

#### Explicação Detalhada do Comando
- **cat change-log.txt**: Como acima, exibe o arquivo de log de mudanças.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Ver detalhes da execução para identificar falhas (ex: cópia de dados pode ter corrompido). Motivo: Logs de mudança complementam a doc.
- **Quando usar?** Em auditorias ou troubleshooting pós-mudança.
- **Motivo e objetivo geral:** Registra ações para accountability; ajuda a replicar ou reverter passos.

Então, o log mostra cópia feita, config alterada para /data/billing, e validação "passou". Mas logs do app dizem corrupção – talvez validação básica não checou integridade. Próximo: confirmar a config atual, para ver se ainda aponta para o diretório corrompido. Usamos `find` se necessário.

### Passo 7: Verificação da Configuração Atual

Beleza, o log confirma a mudança na config. Vamos achar e ler o arquivo de config para verificar se está usando /data/billing – isso liga os logs à GMUD.

#### Comando para Encontrar Configuração (se necessário)
```bash
find / -name "billing-config.yml" 2>/dev/null
```

#### Retorno Esperado (exemplo)
```
/etc/marialazaracloud/billing-config.yml
```

#### Explicação Detalhada do Comando
- **find / -name "billing-config.yml"**: Busca arquivo exato pelo nome.
- **2>/dev/null**: Ignora erros.

Agora, navegamos e lemos.

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

#### Explicação Detalhada do Comando
- **cd /etc/marialazaracloud**: Muda para diretório de configs (/etc é padrão para configs no Linux).
- **cat billing-config.yml**: Exibe o arquivo YAML de configuração.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Verificar se a config aponta para o diretório problemático (/data/billing). Motivo: Configs erradas causam falhas.
- **Quando usar?** Sempre que suspeita de misconfiguração.
- **Motivo e objetivo geral:** Configs controlam comportamento; verificá-las garante alinhamento com expectativas.

Então, sim: data_directory é /data/billing. Isso bate com os logs. Próximo: inspecionar o /data/billing em si, checando permissões e existência, pois se permissões erradas, o app não acessa.

### Passo 8: Investigação do Diretório de Dados

Ok, config aponta para lá. Vamos listar /data e /data/billing para ver se existe, permissões (devops deve ter acesso) e arquivos. Se permissões ok, problema é no conteúdo.

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

#### Explicação Detalhada do Comando
- **ls**: Comando base para "list" (listar conteúdo de diretório).
- **-l**: Flag para formato longo (detalhes como permissões, dono, tamanho).
- **-a**: Flag para "all" (inclui arquivos ocultos, como . e ..).
- **/data/**: Argumento, o diretório a listar.

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

#### Explicação Detalhada do Comando
Mesmo que acima, mas para /data/billing/.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Checar existência, permissões e conteúdo do diretório problemático. Motivo: Permissões erradas podem causar acessos negados.
- **Quando usar?** Para inspecionar arquivos/diretórios em filesystem issues.
- **Motivo e objetivo geral:** Visão rápida do estado; essencial para segurança e depuração.

Então, diretório existe, permissões ok (devops é dono, pode ler/escrever). Arquivos estão lá, mas tamanhos pequenos – suspeito. Próximo: ler o conteúdo dos arquivos para checar integridade.

### Passo 9: Verificação da Integridade dos Dados

Beleza, estrutura ok. Vamos ler os arquivos – se corrompidos, terão garbage em vez de dados reais, confirmando a causa raiz.

#### Comando para Verificar Arquivo de Transações
```bash
cat /data/billing/transactions
```

#### Retorno Esperado
```
ERROR_MIGRATION_INCOMPLETE_DATA_CORRUPTED_2024
BACKUP_REQUIRED_ROLLBACK_RECOMMENDED
```

#### Explicação Detalhada do Comando
- **cat /data/billing/transactions**: Exibe conteúdo do arquivo de transações.

#### Comando para Verificar Customers
```bash
cat /data/billing/customers.txt
```

#### Retorno Esperado
```
MIGRATION_FAILED_DATA_INTEGRITY_COMPROMISED
```

#### Explicação Detalhada do Comando
Mesmo que acima.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Confirmar se dados estão corrompidos (mensagens de erro no lugar de dados reais). Motivo: Logs apontam para corrupção.
- **Quando usar?** Para validar conteúdo de arquivos em suspeitas de integridade.
- **Motivo e objetivo geral:** Detecta alterações indesejadas; crucial para dados sensíveis.

Então, causa raiz encontrada: arquivos corrompidos com mensagens de erro! Isso pode ter acontecido na cópia da GMUD (talvez interrupção ou bug no cp). Próximo: checar os dados originais em /opt/seeds, para confirmar se estão íntegros e viáveis para rollback.

### Passo 10: Verificação dos Dados Originais

Ok, GMUD menciona /opt/seeds como original. Vamos listar e ler para ver se dados bons – se sim, rollback é seguro.

#### Comando para Encontrar Dados Originais (se necessário)
```bash
find / -type d -name "seeds" 2>/dev/null
```

#### Retorno Esperado (exemplo)
```
/opt/seeds
```

#### Explicação Detalhada do Comando
- **find / -type d -name "seeds"**: Busca diretório chamado "seeds".

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

#### Explicação Detalhada do Comando
- **ls -la /opt/seeds/**: Lista detalhes do diretório original.

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

#### Explicação Detalhada do Comando
- **cat /opt/seeds/transactions**: Exibe dados originais.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Verificar se dados antigos estão íntegros para rollback. Motivo: GMUD menciona migração de /opt/seeds.
- **Quando usar?** Em comparações de backups vs. atuais.
- **Motivo e objetivo geral:** Garante fallback seguro; promove resiliência em falhas.

Então, dados originais íntegros! Perfeito para rollback. Próximo: comparar config atual com backup para confirmar a diferença exata.

### Passo 11: Comparação com Backup da Configuração

Beleza, dados bons. Vamos comparar a config atual com o backup para ver exatamente o que reverter – isso evita erros na edição.

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

#### Explicação Detalhada do Comando
- **diff**: Comando base para comparar diferenças entre arquivos.
- **billing-config.yml**: Primeiro argumento, arquivo atual.
- **billing-config.yml.backup**: Segundo argumento, backup.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Confirmar diferenças para guiar rollback. Motivo: Mostra exatamente o que mudou.
- **Quando usar?** Antes de edições ou reversões.
- **Motivo e objetivo geral:** Evita edições cegas; útil para versionamento manual.

Então, diferença só no data_directory – fácil reverter. Próximo: preparar edição segura, com backup extra e ajuste de permissões, pois configs críticas podem ser read-only.

### Passo 12: Preparação para Edição Segura da Configuração

Ok, confirmado. Vamos fazer backup adicional e permitir edição – isso previne perda se algo der errado.

#### Comando para Criar Backup Adicional
```bash
cp billing-config.yml billing-config.yml.pre-rollback
```

#### Explicação Detalhada do Comando
- **cp**: Comando base para "copy" (copiar arquivos).
- **billing-config.yml**: Argumento fonte.
- **billing-config.yml.pre-rollback**: Argumento destino (novo nome).

#### Comando para Verificar Permissões Atuais
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-r--r--r-- 1 devops devops 287 Sep 27 14:35 billing-config.yml
```

#### Explicação Detalhada do Comando
- **ls -la billing-config.yml**: Lista detalhes de um arquivo específico.

#### Comando para Permitir Edição
```bash
chmod 644 billing-config.yml
```

#### Explicação Detalhada do Comando
- **chmod**: Comando base para "change mode" (mudar permissões).
- **644**: Argumento octal: 6 (rw- para owner), 4 (r-- para group), 4 (r-- para others).
- **billing-config.yml**: Arquivo a alterar.

#### Verificação Pós-Alteração
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-rw-r--r-- 1 devops devops 287 Sep 27 14:35 billing-config.yml
```

#### Reflexões
- **Por quê usar aqui?** Objetivo: Preparar edição segura com backup e permissões. Motivo: Evita perda de dados.
- **Quando usar?** Antes de editar arquivos críticos.
- **Motivo e objetivo geral:** Boa prática de segurança; protege contra erros humanos.

Então, pronto para editar. Próximo: executar o rollback alterando a config com nano.

### Passo 13: Execução do Rollback Conforme Documentado

Beleza, backups feitos. Vamos editar para reverter ao /opt/seeds, seguindo o plano da GMUD.

#### Comando
```bash
nano billing-config.yml
```

#### Ação Necessária
Mude `/data/billing` para `/opt/seeds`. Salve com Ctrl+O, Enter. Saia com Ctrl+X.

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

#### Explicação Detalhada do Comando
- **nano**: Comando base, editor de texto simples (fácil para iniciantes).
- **billing-config.yml**: Argumento, arquivo a editar.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Reverter config para diretório íntegro. Motivo: Segue plano de rollback.
- **Quando usar?** Para edições rápidas em texto.
- **Motivo e objetivo geral:** Simples e seguro; ideal para configs sem IDE.

Então, alterado. Próximo: restaurar permissões para proteger a config.

### Passo 14: Restauração de Permissões Seguras

Ok, edição feita. Vamos voltar para read-only para evitar mudanças acidentais futuras.

#### Comando para Restaurar Permissões Restritivas
```bash
chmod 444 billing-config.yml
```

#### Explicação Detalhada do Comando
- **chmod 444**: 4 (r-- para todos níveis: owner, group, others).
- **billing-config.yml**: Arquivo.

#### Verificação da Segurança
```bash
ls -la billing-config.yml
```

#### Retorno Esperado
```
-r--r--r-- 1 devops devops 287 Sep 27 14:40 billing-config.yml
```

#### Reflexões
- **Por quê usar aqui?** Objetivo: Proteger config contra mudanças acidentais. Motivo: Após edição, restaure segurança.
- **Quando usar?** Em arquivos sensíveis pós-alteração.
- **Motivo e objetivo geral:** Reforça princípio de menor privilégio.

Então, config segura. Próximo: reiniciar o app para aplicar a mudança.

### Passo 15: Reinicialização da Aplicação

Beleza, config revertida. Vamos parar o processo antigo e reiniciar para carregar a nova config.

#### Comando para Parar a Aplicação
```bash
ps aux | grep billing
kill 8
```

#### Explicação Detalhada do Comando
- **ps aux | grep billing**: Filtra processo (como em Passo 2).
- **kill**: Comando base para enviar sinal a processo (padrão é TERM, gracioso).
- **8**: Argumento, PID do processo a parar.

#### Comando para Reiniciar
```bash
/usr/local/bin/start-billing-api.sh
```

#### Retorno Esperado
```
Iniciando billing-api em background...
billing-api iniciado com PID 48
```

#### Explicação Detalhada do Comando
- **/usr/local/bin/start-billing-api.sh**: Executa script shell para iniciar o serviço.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Aplicar nova config reiniciando. Motivo: Apps leem config no start.
- **Quando usar?** Após mudanças em serviços.
- **Motivo e objetivo geral:** Garante atualização; comum em deploys.

Então, reiniciado. Próximo: validar nos logs se erros sumiram.

### Passo 16: Validação da Recuperação

Ok, app rodando de novo. Vamos monitorar logs para confirmar sucesso – deve mostrar /opt/seeds e validações ok.

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
(continua com sucessos...)
```

#### Explicação Detalhada do Comando
- **cd ...**: Muda diretório.
- **tail -f app.log**: Monitora logs.

#### Reflexões
- **Por quê usar aqui?** Objetivo: Confirmar sucesso pós-rollback. Motivo: Logs mostram se erro persistiu.
- **Quando usar?** Após fixes para validar.
- **Motivo e objetivo geral:** Fecha o ciclo de troubleshooting.

Então, sucesso! Erros sumiram. Próximo: comunicar e métricas para relatório.

### Passo 17: Comunicação da Resolução

Informe ao time: Rollback feito, sistema ok. Causa: corrupção na migração.

---

## 📊 Métricas de Recuperação

Beleza, para quantificar: vamos contar erros antes e sucessos depois, para provar a fix.

#### Comando para Contar Erros de Corrupção Antes do Rollback
```bash
grep "Data corruption detected" app.log | wc -l
```

#### Retorno Esperado
```
15
```

#### Explicação Detalhada do Comando
- **grep "Data corruption detected" app.log**: Busca frase exata no arquivo.
- **| wc -l**: Pipe para wc (word count); -l conta linhas (número de ocorrências).

#### Comando para Contar Sucessos Pós-Rollback
```bash
grep "Transaction validation completed successfully" app.log | wc -l
```

#### Retorno Esperado
```
8
```

#### Explicação Detalhada do Comando
Mesmo que acima.

#### Comando para Ver Últimas 10 Linhas
```bash
tail -n 10 app.log
```

#### Retorno Esperado
```
(linhas de sucesso...)
```

#### Explicação Detalhada do Comando
- **tail -n 10 app.log**: -n especifica número de linhas (10).

#### Reflexões
- **Por quê usar aqui?** Objetivo: Quantificar antes/depois para relatório. Motivo: Evidência numérica.
- **Quando usar?** Em análises de logs para métricas.
- **Motivo e objetivo geral:** Torna sucesso mensurável; útil para relatórios.

**Lições aprendidas:** Valide dados em migrações. Explore sistemas legados com raciocínio passo a passo.

---

## 🛠️ Conceitos Fundamentais

### 1. Sistema de Permissões Unix
- r: ler, w: escrever, x: executar.
Níveis: owner, group, others. Ex: 644 = owner rw, outros r.

### 2. Gerenciamento de Processos
- `ps aux`: lista.
- `kill PID`: para.

### 3. Análise de Logs
- `tail -f`: ao vivo.
- `grep`: busca.

### 4. Integridade de Dados
Verifique conteúdo para corrupção.

### 5. Change Management
Documente mudanças e rollbacks.

---

## 📝 Glossário

**GMUD:** Processo para mudanças.  
**Rollback:** Voltar ao anterior.  
**PID:** ID de processo.  
**Log Analysis:** Ver logs.  
**Data Corruption:** Dados estragados.  
**Troubleshooting:** Resolver passo a passo.

---

## 🎯 Lições Aprendidas

- Verifique dados em mudanças.
- Documente para ajudar.
- Use raciocínio narrativo para entender o porquê em cada fase.

---

## 📚 Referências e Materiais Adicionais

### Executar o Laboratório
```bash
docker pull marialazaradev/linux-essentials:latest
docker run -it --name devops-investigation marialazaradev/linux-essentials:latest
```

### Próximos Passos
Pratique com narrativas explicativas.

---

## 🎬 Conclusão

Você investigou um incidente em sistema legado, com explicações narrativas em cada fase para entender o fluxo. Isso ajuda a raciocinar melhor em cenários reais.

---

## 👋 Agradecimentos

Criado por Maria Lazara. Inscreva-se: https://www.youtube.com/@marialazaradev

Obrigada! 🚀

---

**Fim do Material Complementar**