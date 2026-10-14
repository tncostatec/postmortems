# 2026-09-13 — Minerador de criptomoeda via RCE em frontend Next.js

## Resumo

Um frontend Next.js 15.1.3 (build standalone, contêiner não-root, único serviço público daquela VPS) foi
explorado por uma cadeia de execução remota de código conhecida da versão. O atacante, como o usuário
sem privilégios do contêiner, baixou um minerador Monero e o copiou para dentro do diretório `.next`.
O processo estourava o limite de 512 MB do contêiner em laço de OOM por quatro dias sem ninguém notar,
até o scanner de malware do provedor parar a VM. Contido e corrigido no mesmo dia: artefatos removidos
em modo de recuperação, framework atualizado, contêiner endurecido (rootfs somente leitura, `tmpfs`
`noexec`, `cap_drop: ALL`, `no-new-privileges`, limite de PIDs), segredo de autenticação rotacionado,
portas de administração fechadas ao firewall e atualização de dependências automatizada. O host, o
banco e o segundo sistema da VPS estavam íntegros.

## Contexto

- VPS Ubuntu hospedando dois sistemas web atrás de um proxy reverso (Traefik), gerenciados por um painel
  de containers. Só o frontend de um deles era exposto publicamente.
- O frontend rodava Next.js 15.1.3 em build standalone, como `uid 1001` (sem root), com limite de
  memória de 512 MB, sem rootfs somente leitura e sem restrição de execução em diretórios graváveis.
- O proxy tinha access log ligado, mas com rotação em 30 MB e sem persistência em arquivo.

## Linha do tempo (UTC)

| Quando | O que | Fonte |
|---|---|---|
| 09/09 03:31 | Execução inicial do payload dentro do contêiner do frontend. Download do minerador para `/tmp/lrt`, cópia para `/app/.next/n0MHSG3`. | `mtime` dos artefatos, log do contêiner |
| 09/09 → 13/09 | Minerador em laço: estoura os 512 MB, é morto pelo OOM, é relançado. Nenhum alerta. | `memory.events` do cgroup, log de OOM |
| 13/09 04:04 | Scanner de malware do provedor (Monarx) para a VM (`ct_stop`) e aplica limites, apontando o processo `/app/.next/n0MHSG3`. | notificação e histórico de ações do provedor |
| 13/09 manhã | Análise com o disco montado em modo de recuperação; identificação da cadeia de exploração nos logs; verificação do host. | sessão no modo de recuperação |
| 13/09 tarde | Correção e endurecimento aplicados e provados. Incidente encerrado. | commits de infra e do app |

## Causa raiz

**Execução remota de código no Next.js 15.1.3** pela cadeia React Flight / Server Actions (advisory
GHSA-9qr9-h5gf-34mp e correlatas). Assinaturas encontradas no log do contêiner: `returnNaN is not
defined`, `NEXT_REDIRECT`, `EACCES ... open 'let'`.

A partir do RCE, o atacante executou como `uid 1001` (o usuário `nextjs` do build standalone):

1. baixou um binário ELF empacotado com UPX (minerador Monero) para `/tmp/lrt`;
2. copiou para `/app/.next/n0MHSG3` e o executou;
3. tentou conectar ao pool/C2 em `216.9.226.120:80` e `94.183.234.121:80`, **sempre com `ETIMEDOUT`**:
   a saída nunca conectou, provavelmente sem lucro para o atacante.

### Fatores contribuintes

1. **Versão vulnerável do framework em produção** sem cadência de atualização: a correção já existia.
2. **Diretórios graváveis e executáveis** dentro do contêiner (`/tmp` e `/app/.next`): o RCE virou
   persistência de processo sem nenhuma barreira adicional.
3. **Sem alerta de OOM-kill.** Um processo batendo no limite de memória milhares de vezes em quatro
   dias deveria ter acordado alguém.
4. **Retenção curta do access log do proxy.** O IP de origem da exploração já tinha sido descartado
   quando o incidente foi percebido.

### O que limitou o estrago

- Contêiner **não-root** e **efêmero** (recriado no deploy): sem chave SSH nova, sem usuário novo, sem
  cron, sem unidade systemd, sem `LD_PRELOAD`, sem binário SUID novo no host; sem movimento lateral para
  o segundo sistema; bancos íntegros. Tudo verificado com o disco montado fora da VM.
- A saída de rede para os endereços do C2 nunca completou.

## Detecção

Pelo provedor, não por mim. O minerador rodou quatro dias dentro do limite de memória do contêiner, sem
derrubar o serviço e sem gerar alerta. A única anomalia visível, o OOM-kill repetido, não era monitorada.

## Contenção e correção

| Medida | Como foi provada |
|---|---|
| **Modo de recuperação** do provedor (via API): disco montado em `/mnt/sdb1`, remoção dos quatro artefatos (`n0MHSG3`, `D9gdBQA`, `/tmp/let`, `/tmp/lrt`), frontend deixado com `restart: no` para não voltar vulnerável, saída do modo de recuperação | varredura do disco montado por hash e por caminho; host sem persistência |
| **Next.js 15.1.3 → 15.5.25** e `next-auth` atualizado | build e deploy; rota pública 200 |
| **Endurecimento do contêiner** (abaixo) | três testes dentro do contêiner em produção: gravar + `chmod +x` + executar em `/tmp` → `Permission denied`; escrever em `/app/.next` → `Read-only file system`; `/app/.next/cache` grava (a aplicação funciona) mas executar lá → `Permission denied` |
| **Segredo de autenticação rotacionado** | contêiner subiu com o segredo novo; sessões antigas invalidadas |
| **Firewall**: portas do painel de containers e do app fechadas para tudo exceto os IPs de administração | de um IP não autorizado a porta dá timeout; 443 segue no ar |
| **Dependabot** nos dois sistemas da VPS, agrupando minor e patch; majors ignorados (um major de CSS quebrava o build e o CI) | PRs abertos e CI verde |
| Segundo sistema da VPS com o framework atualizado por precaução | deploy automático a partir da branch de produção; rota pública 200 |

### O padrão de endurecimento aplicado

Para um serviço que não precisa escrever no rootfs (Next.js standalone só escreve cache):

```yaml
services:
  frontend:
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
      - /app/.next/cache:rw,noexec,nosuid,size=256m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          pids: 200
```

Observações medidas:

- `read_only: true` quebra a aplicação se ela escrever fora dos `tmpfs`. Mapear **todo** caminho
  gravável e validar que o contêiner fica estável (sem laço de restart) e a rota pública responde.
- `pids` em `deploy.resources.limits` e `pids_limit:` juntos conflitam e quebram o Compose; usar um só.
- O minerador usou exatamente os caminhos que o padrão fecha: gravar e executar em `/tmp`, escrever em
  `/app/.next`.

## Lições

1. **Defesa em profundidade para o caso "o RCE vai acontecer".** Rootfs somente leitura + `tmpfs`
   `noexec` + `cap_drop: ALL` + `no-new-privileges` neutralizam o *drop-and-exec* mesmo com execução
   remota de código. É configuração, não custo.
2. **Cadência de atualização é controle de segurança.** Dependabot agrupado + CI como portão. Uma versão
   de framework com advisory público em produção é dívida com juros.
3. **OOM-kill repetido é alerta, não ruído.** Monitorar `memory.events` e avisar na primeira sequência.
4. **Persistir o access log do proxy** com retenção de semanas. Sem ele, a investigação perde a origem.
5. **Não-root e efêmero compraram o resultado.** O comprometimento ficou no contêiner porque o contêiner
   não podia mais nada. Manter assim em todo serviço público.
6. **Egress filtrado.** O C2 nunca conectou por sorte. Com saída restrita por política, teria sido por
   desenho.
7. **Portas de administração nunca ficam abertas ao mundo.** Painel e porta do app só para IPs
   conhecidos; a rede pública vê 443.

## Indicadores de comprometimento

| Tipo | Valor |
|---|---|
| sha256 do minerador | `e7ed560665ec93e02779d4b62db73ff7914b1b631330ce31df756e41047af4c7` |
| Caminhos | `/tmp/lrt`, `/tmp/let`, `/app/.next/n0MHSG3`, `/app/.next/D9gdBQA` |
| Rede | `216.9.226.120:80`, `94.183.234.121:80` |
| Assinaturas no log do Next.js | `returnNaN is not defined`, `NEXT_REDIRECT`, `EACCES ... open 'let'` |

## Checklist para ambientes parecidos

- [ ] Qual versão do framework está em produção e quando foi atualizada pela última vez?
- [ ] O contêiner público roda como não-root, com rootfs somente leitura e `tmpfs` `noexec`?
- [ ] Existe alerta de OOM-kill por contêiner?
- [ ] O access log do proxy sobrevive uma semana?
- [ ] Portas de painel e de app estão fechadas ao mundo?
- [ ] A saída de rede do contêiner é restrita?
