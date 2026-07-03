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

### [REVISAR] EC2 Placement Groups — 2026-07-02

Placement group = você dizendo à AWS **como posicionar fisicamente** as instâncias. 3 estratégias:

| Estratégia | Posicionamento | Palavra-chave na prova |
|-----------|----------------|------------------------|
| **Cluster** | Todas juntas, mesmo rack, mesma AZ | "low latency", "high network throughput", HPC |
| **Spread** | Cada uma em hardware distinto (máx. 7/AZ) | "reduce risk of simultaneous failure" |
| **Partition** | Grupos em racks separados | Hadoop, Kafka, Cassandra |

**Analogia:** cluster = carros lado a lado na mesma fileira; spread = um por andar; partition = vans por setor.

**Insufficient capacity error** ao ADICIONAR instância num cluster group = o rack encheu. **Solução: stop/start de TODAS as instâncias do grupo** — a AWS realoca o grupo inteiro para onde todas caibam juntas.

**Prevenção:** lançar todas as instâncias num único launch request, mesmo instance type.

**Pegadinha:** "limite de 12 instâncias por placement group" NÃO existe (opção inventada, mesma família de NAT Multi-AZ e DAX multi-região).

**Domínio:** Design High-Performing Architectures (24%)
**Confiança atual:** Baixo

---

### Redshift ≠ Redis + Escala de Latências — 2026-07-02

**Nomes parecidos, serviços opostos:** Red**shift** = data warehouse OLAP em disco (queries analíticas, sub-segundo na melhor hipótese). Red**is** (ElastiCache) = cache in-memory (microssegundos).

**Escala de latência para decorar:**
Redis/DAX (microssegundos) → DynamoDB (single-digit ms) → Aurora/RDS (dezenas de ms) → Redshift (sub-segundo+)

**Pipeline IoT canônico:** Kinesis (ingere o stream) → Lambda (processa) → DynamoDB (serve em ms).

**Domínio:** Design High-Performing Architectures (24%)
**Confiança atual:** Médio

---

### SQS Standard duplica POR DESIGN — 2026-07-02

Duas causas de mensagem duplicada:
1. Visibility Timeout expira no meio do processamento → **corrigível** aumentando o timeout
2. Infraestrutura do SQS Standard entrega 2x por design (**at-least-once**) → **nenhuma configuração impede**

**"Processar exatamente uma vez" → SQS FIFO** (exactly-once + deduplicação por conteúdo). Visibility Timeout ajusta *quando* a mensagem reaparece, nunca *se* ela duplica.

**Domínio:** Design Resilient Architectures (26%)
**Confiança atual:** Baixo (errado 2x — revisar antes da prova)

---

### Direções de custo de transferência + StackSets — 2026-07-02

**Hotel AWS:** entrar (internet → AWS) = grátis. Sair (AWS → internet) = paga (~$0,09/GB). Mesma região S3↔EC2 = grátis.

**NAT Gateway cobra ~$0,045/GB processado** — tráfego S3 por NAT é desperdício puro; **Gateway Endpoint** (grátis) elimina.

**CloudFormation StackSets** = mesmo template implantado em **múltiplas contas e regiões de uma vez**. Palavra-chave: "across accounts" / "toda a organização".

**Domínio:** Design Cost-Optimized Architectures (20%)
**Confiança atual:** Médio

---

### Redis vs Memcached — 2026-07-02

| | Redis | Memcached |
|--|-------|-----------|
| Threads | Single-threaded | **Multithreaded** ← palavra-chave exclusiva |
| Estruturas | Sorted Sets, listas, pub/sub | Key-value simples |
| Persistência/réplicas/failover | ✅ | ❌ (mas **Auto Discovery** detecta e substitui nós) |

**Truque:** "multithreaded" → Memcached. "Persistência/Sorted Sets/failover" → Redis. **Global Datastore** = replicação de cache entre regiões (réplicas remotas read-only) — não serve para sessões com escrita.

**Domínio:** Design High-Performing Architectures (24%)
**Confiança atual:** Médio

---

### Golden Rules — Simulado Tutorials Dojo (erradas) — 2026-07-02

1. **Lambda Function URL**: webhook HTTP direto para Lambda, sem API Gateway. "Most operationally efficient + webhook" → Function URL. SQS não aceita HTTP POST externo.
2. **IAM User via CLI/API nasce SEM credenciais**: chamadas de API exigem **Access Keys** + permissões. Console = escolhe senha/keys na criação.
3. **Portas**: SSH 22 | **RDP 3389** | MySQL 3306 | PostgreSQL 5432. "Remote Desktop não conecta" → inbound 3389 no SG.
4. **Duração mínima S3**: Standard = nenhuma | IA = 30d | Glacier = 90d | Deep Archive = 180d. Dado temporário (horas) → **S3 Standard** (deletar antes do mínimo paga o período inteiro).
5. **CloudFront origin failover** → **origin group** com 2 origins (ex.: 2 EC2 em AZs distintas). ASG não é origin; S3 não serve dinâmico.
6. **Decoupling 3 camadas**: estático → S3 | app → ECS + Service Auto Scaling | banco → RDS Multi-AZ. Lambda não roda long-running (>15min); CloudFront não hospeda.
7. **Governança multi-conta**: Organizations + Consolidated Billing (custo central) + **IAM cross-account roles** (admin sem criar usuários). VPC/AZ separadas ≠ autonomia de conta.
8. **GPS/telemetria em tempo real + múltiplos consumers** → Kinesis. EMR = batch; AppStream = streaming de desktop (distrator); SQS = 1 consumer.
9. **HPC + integração nativa S3 + POSIX** → **FSx for Lustre** (EFS não integra com S3).
10. **"Reserve capacity in a specific AZ" sem compromisso** → **On-Demand Capacity Reservation**. Regional RI NÃO reserva capacidade (só desconto). Capacidade ≠ desconto.

**Confiança atual:** Médio (revisar antes da prova)

---

### [REVISAR] Re-Quiz 2026-07-02 — Resultado 18/20 (90%)

**Fixados no re-teste** (errados antes, certos agora): SQS FIFO exactly-once, SSM Session Manager, Aurora Global + Route 53, DynamoDB Global Tables, io2 Block Express, RDS Proxy + reserved concurrency, IAM Identity Center, placement groups, Beanstalk, Lambda Function URL, ZSET, cadeia CloudTrail→CW Logs→filter→alarme→SNS, duração mínima S3, Access Keys, RDP 3389, Egress-only IGW, EBS Encryption by Default, FSx Lustre.

**Os 2 que persistem — revisar antes da prova:**

1. **EFS Bursting vs Provisioned:** Bursting = poupança de créditos — acumula quando o uso está ABAIXO do baseline (50 MB/s por TB). Demanda **24/7 contínua** = créditos nunca recarregam = throttling permanente no baseline. Pergunta certa: "a demanda tem folga para recarregar créditos?" Se não → **Provisioned**. Não avaliar se o baseline "parece suficiente".

2. **Capacity Reservation vs Savings Plan/RI:** capacidade e desconto são EIXOS INDEPENDENTES. SP e Regional RI = instrumentos financeiros, **nunca garantem capacidade**. "Não pode faltar capacidade na AZ X" + "sem compromisso de 1/3 anos" → **On-Demand Capacity Reservation** (cria/cancela livremente, sem desconto). Zonal RI garante capacidade MAS exige compromisso.

**Padrão dos dois erros:** escolher o benefício que ninguém pediu (baseline "razoável", desconto não solicitado). Responder ao requisito literal.

**Domínio:** High-Performing (1) + Cost-Optimized (2)
**Confiança atual:** Baixo nos 2 persistentes, Alto nos demais

---

### [REVISAR] Mini-Simulado Cost-Optimized (calibração nova) — 6/9 (67%) — 2026-07-02

Nível agora pareado com Tutorials Dojo (64% lá, 67% aqui). Erradas:

1. **DynamoDB DAX vs On-Demand:** DAX = cache de LEITURA (resolve latência/leitura repetida). Não resolve throttling de ESCRITA nem custo de ociosidade (cluster cobra 24/7). Tráfego imprevisível + picos súbitos + ociosidade cara → **modo On-Demand** (escala instantânea, paga por requisição).
2. **gp2 acopla IOPS ao tamanho (3 IOPS/GB):** encolher gp2 = perder IOPS. Volume gp2 grande "por causa das IOPS" → migrar para **gp3** (3.000 IOPS de baseline inclusas em qualquer tamanho + ~20% mais barato/GB). Otimização de EBS mais cobrada da SAA.
3. **S3 Requester Pays:** "quem baixa deve pagar o próprio download" (datasets compartilhados com parceiros) → Requester Pays (requisitante autenticado paga transferência+requests; dono paga só storage). Presigned URL = controle de ACESSO, não transferência de CUSTO.

**Acertadas (fixadas):** right-size antes de Savings Plan; Gateway Endpoint vs Interface (menor esforço/custo dominante); stop/start p/ dev com estado; CloudFront p/ egress de estático; Spot em EMR (master On-Demand); Fargate p/ containers intermitentes com baixa utilização.

**Domínio:** Design Cost-Optimized Architectures (20%)
**Confiança atual:** Médio

---
