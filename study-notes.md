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

### CloudFront múltiplos origins (Static + Dynamic) — 2026-07-02

Uma distribuição CloudFront pode ter **múltiplos origins** — não precisa de duas soluções separadas.

- **S3 como origin** → conteúdo estático cacheado na borda
- **ALB como origin** → conteúdo dinâmico via rede backbone AWS

Cache behaviors definem qual origin atende cada path (`/static/*` → S3, `/*` → ALB).

**Armadilha de prova:** Global Accelerator **não aceita S3 como endpoint** (só EC2, ELB, Elastic IP, Lambda). Opções que combinam Global Accelerator + S3 são sempre erradas.

**Truque:** Resposta simples (CloudFront com S3 + ALB) bate resposta complexa (Global Accelerator + CloudFront separados). Na SAA-C03, complexidade desnecessária = errado.

**Domínio:** Design High-Performing Architectures (24%)
**Confiança atual:** Médio

---

### IAM Identity Center + Active Directory — 2026-07-02

**Federação:** a AWS confia no AD corporativo em vez de duplicar usuários. Fluxo:

```
Credencial corporativa → AD valida → AD Connector (ponte) → IAM Identity Center → Console AWS
```

- **Active Directory** = cadastro central de funcionários da empresa (on-premises)
- **AD Connector** = proxy que encaminha autenticação para o AD on-premises
- **AWS Directory Service** = infraestrutura de diretório — sozinho NÃO dá acesso ao console
- **IAM Identity Center** = quem efetivamente concede acesso SSO ao console, sem criar usuário IAM

**Truque de prova:** "credenciais corporativas/AD + console AWS + sem usuários IAM" → IAM Identity Center (ex-AWS SSO).

**Técnica de eliminação:** procurar contradição direta entre opção e requisito — ex.: opção diz "criar usuários IAM automaticamente" quando o requisito é "sem criar usuários IAM". A opção se auto-elimina, sem precisar conhecer a tecnologia.

**Técnica de eliminação:** procurar contradição direta entre opção e requisito — ex.: opção diz "criar usuários IAM automaticamente" quando o requisito é "sem criar usuários IAM". A opção se auto-elimina, sem precisar conhecer a tecnologia.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Alto

---

### Camadas de Proteção: SG vs NACL vs WAF — 2026-07-02

**Regra fundamental:** SG e NACL protegem recursos DENTRO da sua VPC. WAF protege serviços gerenciados de borda FORA dela.

- **Security Group** (nível ENI/instância): EC2, RDS, ALB/NLB, Lambda em VPC, ElastiCache. Só ALLOW, stateful. **Não existe em:** API Gateway público, CloudFront, S3.
- **NACL** (nível subnet): único lugar de rede da VPC com DENY. Não alcança endpoints públicos de serviços gerenciados.
- **WAF** (Layer 7): só acopla em **CloudFront, ALB, API Gateway REST e AppSync** (nunca NLB!). Faz o que SG/NACL não fazem: IP set (block), rate-based rule por IP, SQLi/XSS, geo-blocking.

**Tabela de decisão:**

| Cenário | Resposta |
|---------|----------|
| Bloquear IP em EC2/subnet | NACL (DENY) |
| Bloquear IP em API GW/CloudFront/ALB | WAF (IP set) |
| Rate limiting por cliente | WAF (rate-based) |
| SQLi/XSS | WAF |
| DDoS volumétrico L3/L4 | Shield |
| "WAF no NLB" | Pegadinha — não existe |

**Técnica geral:** em Select com 2 requisitos, eliminar por cobertura — a certa atende os dois, distratores atendem um ou nenhum.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Médio

---

### [REVISAR] Bastion Host vs SSM Session Manager — 2026-07-02

**Bastion host (jump box):** servidor em subnet pública usado como "ponte" SSH para instâncias privadas. Problemas clássicos: porta 22 exposta à internet + nenhum registro do que o admin faz na sessão.

**SSM Session Manager:** substitui o bastion por completo. O SSM Agent na instância conecta **para fora** ao serviço SSM:
- **Nenhuma porta de entrada** (nem 22, nem bastion, nem IP público)
- Autenticação via **IAM** (MFA, permissão por instância)
- **Grava cada comando digitado** no S3/CloudWatch Logs

**Pegadinha central:** CloudTrail registra **chamadas de API AWS**, nunca o conteúdo de sessões SSH/shell. "Registro de sessões" = comandos digitados → Session Manager logging, não CloudTrail.

**Técnica de verbo:** requisito diz "eliminar" → opção que apenas **reduz** (ex.: restringir SG a IPs corporativos — porta 22 ainda existe) está errada por definição.

**Truque de prova:** "bastion / porta 22 / SSH + auditoria de sessões + menor complexidade" → SSM Session Manager.

**Domínio:** Design Secure Architectures (30%)
**Confiança atual:** Baixo

---
