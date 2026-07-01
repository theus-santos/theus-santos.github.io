# Notas de Estudo — AWS Certified Solutions Architect Associate (SAA-C03)

> Arquivo gerado automaticamente pelo AWS Study Buddy. Use `#save-note` para adicionar notas e `#get-notes` para consultar.

---

### Security Groups vs NACLs — 2026-07-01

**Security Groups:** stateful — basta uma regra inbound; o retorno é automático. Somente ALLOW. Aplicado no nível da ENI (instância).

**NACLs:** stateless — precisa de regra inbound E outbound explícita. Suporta ALLOW e DENY. Aplicado no nível da subnet. Regras avaliadas em ordem numérica crescente (a primeira que bate ganha).

**Truque de prova:** Se a questão fala em "bloquear um IP específico" → NACL (único jeito de fazer DENY). Se fala em "menor privilégio por instância" → Security Group.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Médio

---

### DR Strategies — 2026-07-01

Ordem crescente de custo e decrescente de RTO/RPO:

1. **Backup & Restore** — mais barato, RTO/RPO mais alto (horas). Dados no S3, restore manual.
2. **Pilot Light** — componentes críticos sempre ligados (DB replicado), resto desligado. RTO: minutos/horas.
3. **Warm Standby** — versão reduzida do ambiente sempre ativa. Scale up no DR. RTO: minutos.
4. **Multi-Site Active-Active** — tráfego dividido entre regiões em tempo real. RTO/RPO ~0. Mais caro.

**Truque de prova:** Palavras "menor custo" → Backup & Restore. "RTO < 15min" → Warm Standby ou Multi-Site. "RPO ~0" → Multi-Site.

**Domínio:** Design Resilient Architectures (26%)
**Confiança atual:** Alto

---

### KMS CMK Cross-Account — 2026-07-01

Para acesso cross-account a dados criptografados com KMS CMK:
1. **Key Policy** da CMK precisa permitir a conta destino (não basta IAM policy na conta origem).
2. **IAM Policy** na conta destino precisa ter `kms:Decrypt` + `kms:GenerateDataKey`.
3. **S3 Bucket Policy** (se S3) precisa permitir a conta destino também.

**Truque de prova:** Dois passos obrigatórios — Key Policy na conta do KMS + IAM na conta que acessa.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Alto

---

### aws:PrincipalOrgID vs aws:PrincipalOrgPaths — 2026-07-01

- `aws:PrincipalOrgID` → condição que aplica a **toda a organização** (qualquer conta membro).
- `aws:PrincipalOrgPaths` → condição que aplica a **OUs específicas** dentro da organização.

**Caso de uso na prova:** S3 Bucket Policy que permite acesso apenas a contas da mesma org → usar `aws:PrincipalOrgID` no `Condition` block. É mais simples e não precisa listar cada account ID.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Alto

---

### Kinesis Data Streams vs SQS — 2026-07-01

| | Kinesis | SQS |
|--|---------|-----|
| Consumidores | Múltiplos simultâneos | Um por mensagem (por padrão) |
| Ordenação | Por shard (garantida) | Melhor esforço (FIFO opcional) |
| Replay | Sim (até 7 dias) | Não (mensagem deletada após consumo) |
| Caso de uso | Analytics em tempo real, múltiplos consumidores | Desacoplamento, processamento único |

**Truque de prova:** "múltiplos consumidores independentes processando o mesmo stream" → Kinesis. "fila de tarefas, um worker por item" → SQS.

**Domínio:** Design High-Performing Architectures (24%)
**Confiança atual:** Alto

---
