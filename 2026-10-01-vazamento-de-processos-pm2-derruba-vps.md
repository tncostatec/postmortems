# 2026-10-01 — Contêiner com PM2 vaza processos, VPS inteira vai a 100% de CPU e o provedor limita a máquina

## Resumo

Um agendador de redes sociais auto-hospedado (Postiz, stack própria em Docker Compose na mesma VPS de um
SaaS) entrou em laço de reinício depois de um reboot de manutenção. A imagem oficial da época não trazia
o utilitário `ps`; o PM2, que gerencia os processos dentro do contêiner, usa `ps` para matar a árvore de
processos a cada restart. Sem `ps`, cada restart matava só o processo pai e deixava o aplicativo Node
órfão. Em 16 horas foram 13 mil restarts, 1.611 processos, 4 GB de memória cheios, 17.619 OOM-kills e
cerca de 28 horas de CPU consumidas. Às 04:02 o provedor aplicou um limite de CPU à VM inteira; às 08:00
a API do SaaS entrou em laço de morte de workers e saiu do ar. Contenção: parar o stack do agendador e
reduzir a API a um worker. Correção: imagem própria com `procps`, teto de CPU em todo contêiner, vigia
com disjuntor e um vigia de recursos do host com alerta por caminho independente.

## Contexto

- VPS Ubuntu com Docker Compose. Stack principal: API Python (uvicorn, 4 workers), PostgreSQL, Redis,
  MinIO, frontend Next.js, Nginx, n8n e certbot. Stack secundário, em outro diretório: o agendador
  (Postiz), com seus próprios Postgres, Redis e Temporal.
- Os contêineres do stack principal já tinham limites de memória. O agendador **não tinha teto de CPU**.
- Dias antes, um "orchestrator travado" no agendador vinha sendo tratado com um vigia que rodava
  `pm2 restart` a cada 10 minutos, sem limite de tentativas. Esse vigia é personagem deste incidente.

## Linha do tempo (horário local, UTC−3)

| Quando | O que | Fonte |
|---|---|---|
| 30/09 16:18 | Reboot manual para subir o kernel. O agendador volta em crash-loop. CPU da VM sai de 5–14 % para 20–36 %. | métricas do provedor, `sar` |
| 30/09 noite | O vigia de 10 min reinicia o orchestrator repetidamente. Cada restart deixa um `node main.js` órfão. | `pm2 list` (↺ na casa dos milhares), `docker top` |
| 01/10 00:30 | Kernel a 70–80 %: a memória do cgroup do agendador está cheia e o kernel passa a gastar CPU em OOM thrash. | `sar -u`, `memory.events` do cgroup |
| 01/10 00:30–04:00 | VM a ~100 % de CPU por três horas e meia. | métricas do provedor |
| 01/10 04:02 | Provedor aplica `ct_set_limits` (limitação de CPU da VM), reaplicado às 05:03, 06:03 e 07:02. O *steal* sobe de 25 % para 80–90 %. | histórico de ações da API do provedor |
| 01/10 ~08:00 | A API do SaaS entra em laço: o supervisor do uvicorn mata workers que não respondem ao ping em 5 s, e sob CPU estrangulada um worker novo levava mais que isso só para importar a aplicação. App fora do ar. | log da API: `Waiting for child process` / `Child process died`, sem traceback |
| 01/10 ~08:30 | Diagnóstico: `docker stats`, `pm2 list`, contagem de PIDs, `memory.events`. Causa identificada. | sessão SSH |
| 01/10 manhã | Contenção: `docker compose stop` no agendador. API com `WEB_CONCURRENCY=1` (um worker, sem supervisor) e backend recriado. SaaS volta, degradado. | compose, `.env` |
| 01/10 09:54 | Provedor remove a limitação depois do pedido. | ações da API do provedor |
| 01/10 até 10:30 | Imagem própria do agendador, limites de CPU, vigia com disjuntor, vigia de recursos. API de volta a 4 workers. Tudo provado (abaixo). | commits de infra |

## Causa raiz

**O PM2 mata a árvore de processos executando `ps -e -o pid=,ppid=`** (`pm2/lib/TreeKill.js`). A imagem
oficial do agendador (Debian slim) não trazia `procps`. O `spawn` do `ps` falhava em silêncio: morria só o
filho direto (`pnpm`) e o `node main.js` ficava órfão, vivo e consumindo memória. Com a aplicação em
crash-loop, cada ciclo de restart somava um processo.

Prova: `pgrep -fc main.js` crescia a cada `pm2 restart`; `pm2 list` mostrava milhares de restarts;
`memory.events` do cgroup com `oom_kill` em 17.619.

### Fatores contribuintes

1. **Vigia sem disjuntor.** O `pm2 restart` a cada 10 minutos, criado dias antes para o "orchestrator
   travado", transformou um vazamento lento num vazamento rápido. O travamento era o próprio crash-loop.
2. **Sem teto de CPU no agendador.** Um stack secundário conseguiu consumir a VM inteira e derrubar o
   principal.
3. **Supervisor do uvicorn com ping fixo de 5 s** (uvicorn 0.30+, sem opção de CLI para mudar). Sob CPU
   estrangulada, a própria proteção vira o modo de falha: mata o worker antes de ele terminar de subir.
4. **Alerta compartilhando destino com o host.** O vigia diário das 07:30 (workflow n8n na mesma VPS)
   falhou ao enviar o aviso por WhatsApp porque a instância do WhatsApp estava reconectando no mesmo
   host degradado.

## Detecção

Tardia. A percepção veio pelo SaaS fora do ar, horas depois do pico. As métricas do provedor mostravam
a subida desde as 16:18 do dia anterior, mas não havia alerta de CPU sustentada, de *steal* nem de
OOM-kill. O sintoma dos dias anteriores ("orchestrator travado") foi tratado com reinício em vez de
investigado.

## Contenção e correção

**Contenção**

- `docker compose stop` no stack do agendador: estancou o consumo imediatamente.
- API com `WEB_CONCURRENCY=1`: com um único worker não existe supervisor, logo não há ping de 5 s. O
  SaaS voltou degradado, mas voltou, ainda sob a limitação do provedor.

**Correção**

| Medida | Como foi provada |
|---|---|
| Imagem própria do agendador: `FROM` na imagem oficial por **digest fixo** + `apt-get install procps` | `docker exec <c> which ps` → `/usr/bin/ps`; teste de `pm2 restart`: orchestrator antigo morreu (memória do contêiner 2,7 GB → 0,9 GB), um único `node` depois, worker novo registrado nas filas |
| `cpus:` em **todos** os serviços dos dois stacks (1.0 para o agendador); `restart: unless-stopped` | `docker inspect` dos limites; VM estável com o stack de volta: pico de 2,8 GB, 244 PIDs, `oom_kill` = 0, restarts PM2 = 0 |
| Vigia do agendador com **disjuntor**: no máximo 3 restarts em 6 h, depois e-mail e para; `flock` contra execução concorrente; **sem resposta do Temporal = não age** | execução manual nos três ramos (saudável, travado, disjuntor aberto) |
| Vigia de recursos do host (cron a cada 5 min): CPU ≥ 60 % por 15 min, *steal* ≥ 20 % por 10 min, qualquer OOM-kill → e-mail por provedor transacional, caminho independente do WhatsApp | provado ponta a ponta no mesmo dia: alerta às 09:50 com *steal* de 57 %, "resolvido" às 10:05 |
| Sonda da porta do backend do agendador: age só em `ECONNREFUSED`, com o mesmo disjuntor | o backend subiu **travado** (futex, 0 s de CPU, porta fechada) com o contêiner "healthy"; `pm2 restart backend` resolveu em 35 s; a sonda cobre esse caso |
| API de volta a 4 workers depois que o provedor removeu a limitação | 0 mortes de worker nas horas seguintes |

## Efeitos colaterais da resposta

- O teste de `pm2 restart` do orchestrator às 10:00 matou a publicação de um post agendado para as
  08:00 (a fila tinha `maximumAttempts = 1`). Foi republicado à mão pelo mesmo registro depois de
  confirmar na API da rede social que não tinha saído. Lição: não testar reinício na janela de
  publicação, e conhecer a política de retentativa da fila antes de reiniciar o que a consome.
- Dois posts de outras redes saíram com duas horas de atraso.

## Lições

1. **Todo contêiner nasce com teto de CPU e de memória.** Um serviço secundário nunca pode levar a VM
   junto. Limite de memória sem limite de CPU não protege: foi o OOM thrash que comeu a CPU.
2. **Todo vigia que reinicia algo nasce com disjuntor** (N tentativas por janela → alerta e para), trava
   contra concorrência e a regra "sem resposta do serviço de apoio = não age". Reinício sem limite é
   combustível.
3. **Contêiner com PM2: conferir `which ps`.** Se não houver, derivar a imagem com `procps` e fixar o
   digest. Provar o restart contando processos antes e depois.
4. **`Child process died` sem traceback → olhar CPU e *steal* do host antes de caçar bug no código.** O
   ping de 5 s do supervisor do uvicorn não é configurável; `WEB_CONCURRENCY=1` é o modo de emergência.
5. **Alerta não pode compartilhar destino com o que ele vigia.** E-mail por provedor externo, não
   WhatsApp pela instância que mora no mesmo host.
6. **Sintoma recorrente é incidente em curso, não rotina.** "Travou de novo, reinicia" três dias seguidos
   era o vazamento acontecendo.
7. **Alertar em CPU sustentada, *steal* e OOM-kill**, não só em "serviço fora do ar". O provedor viu a
   anomalia oito horas antes de agir; eu deveria ter visto antes dele.

## Checklist para ambientes parecidos

- [ ] `docker compose config` mostra `cpus` e `memory` em todos os serviços?
- [ ] Imagens de terceiros com gerenciador de processos: `which ps` dentro do contêiner?
- [ ] Algum script seu reinicia algo em loop? Onde está o disjuntor?
- [ ] Existe alerta de OOM-kill (`memory.events`) e de *steal*? Por qual caminho ele chega se o host
      estiver degradado?
- [ ] O que acontece com a sua API quando a CPU é estrangulada: ela degrada ou entra em laço?
