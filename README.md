# Postmortems

Relatos de incidentes reais em infraestrutura que opero (VPS Linux com Docker Compose, aplicações Python e
Next.js, bancos PostgreSQL), escritos no formato que uso internamente: linha do tempo, causa raiz, o que
foi feito, como foi provado e o que muda daqui para a frente.

Tudo aqui foi **anonimizado**: nomes de host, IPs das máquinas, domínios, clientes e qualquer dado de
negócio foram removidos. Os indicadores de comprometimento (hashes, endereços de C2) ficam, porque são
úteis a quem investiga a mesma família de ataque. Os números de impacto são os medidos na época.

Postmortem é sem culpa. O objetivo é que o mesmo erro não se repita, nem comigo nem com quem ler.

| Data | Incidente | Classe | Tempo até conter |
|---|---|---|---|
| 2026-10-01 | [Contêiner com PM2 vaza processos, VPS inteira vai a 100% de CPU e o provedor limita a máquina](2026-10-01-vazamento-de-processos-pm2-derruba-vps.md) | Disponibilidade | ~4 h depois do pico; 16 h desde o início do vazamento |
| 2026-09-13 | [Minerador de criptomoeda via RCE em frontend Next.js](2026-09-13-minerador-via-rce-nextjs.md) | Segurança | Mesmo dia; 4 dias desde a execução inicial |

## Formato

1. **Resumo** em quatro linhas: o que caiu, por quanto tempo, causa, correção.
2. **Contexto** mínimo para entender o ambiente.
3. **Linha do tempo** com horários e a fonte de cada observação (métrica, log, comando).
4. **Causa raiz** e fatores contribuintes, separados.
5. **Detecção**: como foi percebido e por que não foi antes.
6. **Contenção e correção**, com o que foi provado e como.
7. **Efeitos colaterais** da própria resposta (também contam).
8. **Lições** acionáveis, escritas como regra ("todo contêiner nasce com teto de CPU"), não como intenção.
9. **Checklist** para quem tem um ambiente parecido.

## Autor

Tiago Costa · [TN Costa Tecnologia](https://www.tncostatecnologia.com.br/) · [LinkedIn](https://www.linkedin.com/in/tiago-costa-7470587a)
