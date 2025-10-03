# Objetivo

Manter Azure VMs sob Defender for Servers Plan 2 (P2) na assinatura, enquanto servidores Azure Arc (e qualquer outro recurso marcado) ficam em Free/sem plano no nível do recurso, usando TAG + Azure Policy.  
Tudo que “subir” no RG `rg-azurearc-...` não recebe Defender for Servers (nem P1 nem P2) → fica Free/sem plano.

---
# Pre-Checks (ARG) — Azure Arc + Defender for Servers (Free por TAG / VMs Azure em P2)

Use **estas queries** no **Azure Resource Graph Explorer** **antes** de aplicar as políticas.  
Objetivo: mapear servidores Arc, TAGs, extensões MDE, planos do Defender (resource-level vs subscription-level) e estimar impacto.

---

## Índice
- [0) Parâmetros (ajuste rápido)](#0-parâmetros-ajuste-rápido)
- [1) Inventário de Azure Arc (todas as subscriptions)](#1-inventário-de-azure-arc-todas-as-subscriptions)
- [2) Arc com/sem TAG (RG alvo)](#2-arc-comsem-tag-rg-alvo)
- [3) Extensão MDE (Windows/Linux) no RG alvo](#3-extensão-mde-windowslinux-no-rg-alvo)
- [4) Defender for Servers — Plano (resource-level vs subscription-level)](#4-defender-for-servers--plano-resourcelevel-vs-subscriptionlevel)
- [5) Auto-provisioning do Defender for Cloud](#5-auto-provisioning-do-defender-for-cloud)
- [6) Policy assignments no RG alvo](#6-policy-assignments-no-rg-alvo)
- [7) Impacto previsto ao aplicar a TAG no RG](#7-impacto-previsto-ao-aplicar-a-tag-no-rg)
- [8) Métricas executivas](#8-métricas-executivas)

---

## 0) Parâmetros (ajuste rápido)

> **Edite aqui** os valores do RG e da TAG. Em seguida, rode as queries das seções abaixo.

```kusto
let rg       = "rg-azurearc-itsbx-us";   // RG alvo (Arc)
let tagName  = "DefenderForServers";     // Nome da TAG de controle
let tagValue = "Disabled";               // Valor da TAG de controle



## Passo 0 — Entender o mecanismo (oficial)


O Defender for Servers pode ser habilitado/desabilitado por recurso (VM/Arc/VMSS), além do nível de assinatura.  
A documentação afirma:  
> “Embora recomendemos habilitar por assinatura, você pode habilitar no nível do recurso…”  
[Microsoft Learn - Tutorial Enable Servers Plan](https://learn.microsoft.com/en-us/azure/defender-for-cloud/tutorial-enable-servers-plan)

Há uma policy *built-in* que desabilita o Defender for Servers para recursos que tenham uma tag específica (vale para VM, VMSS e Arc Machines).  
Referência: [Policy Reference - Azure Arc](https://learn.microsoft.com/en-us/azure/azure-arc/servers/policy-reference)

Para garantir que todos os recursos do RG recebam a tag automaticamente, use a policy built-in:  
**“Inherit a tag from the resource group (if missing)”**  
[Tag Policies - Azure Resource Manager](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies?)

---

## Passo 1 — Coloque a tag no Resource Group

No RG `rg-azurearc-...`:

- Navegue até **Tags** → **Add tag**
- **Key:** `DefenderForServers`
- **Value:** `Disabled`
- Salve.

![Add tag no RG](https://github.com/user-attachments/assets/6c7d2893-6fec-4e45-b10b-c70f988cbf62)

(Depois vamos herdar essa tag para todo recurso criado no RG.)  
Docs de políticas de tag: [Tag Policies - Azure Resource Manager](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies?)

> **Dica**: Se você não quer sensor MDE nos Arc, combine com o time responsável para offboarding do MDE e remover a extensão MDE nas máquinas Arc.

---

### 1. Crie/eleja o Resource Group para os servidores Arc

No portal, vá em **Resource groups → Create** (se já existir, pule).

- **Resource group**: `MyRgsLabAzPolArc` (ou o nome do seu RG)
- **Region**: a mesma que você vai usar nos Arc (ex.: East US)
- **Review + create** → **Create**

---

### 2. Coloque a TAG no Resource Group (para herdar nos recursos)

Abra o seu RG (ex.: `rg-azurearc-itsbx-us`).

- **Menu lateral Tags → Add:**
    - **Name:** `DefenderForServers`
    - **Value:** `Disabled`
- **Save**

> **Dica:** essa TAG no RG será herdada pelos recursos com a policy do próximo passo.

---

### 3. Policy 1 — “Herdar TAG do RG se estiver faltando”

Portal: **Policy → Definitions →** campo de busca:  
`Inherit a tag from the resource group if missing`

- Abra a definição e clique em **Assign**.

#### Basics:
- **Scope**: selecione o Resource Group onde estão/estarão os Arc.
- **Assignment name**: por exemplo `RG-ARC-InheritTag-DefenderForServers`.

![Assign policy inherit tag](https://github.com/user-attachments/assets/3fc81f27-673c-49a5-982c-6323dcdfe958)

#### Parameters:
- **Tag Name:** `DefenderForServers`

![Tag name param](https://github.com/user-attachments/assets/861c8bc3-b675-4e04-93ba-69bdab8274fe)

#### Remediation:
- Marque **Create a remediation task**.
- **Managed identity:** deixe `System assigned managed identity`.
- **Location:** a mesma do RG (ex.: East US).

![Remediation inherit tag](https://github.com/user-attachments/assets/9af76b4f-9954-4a14-b8fd-614cd921566c)

- **Review + create → Create**

> **O que esperar**: recursos novos e existentes do RG receberão a TAG do RG (caso não tenham).

![TAG herdada](https://github.com/user-attachments/assets/cc0ba65a-a446-487f-9d75-7b51b5c2effd)

---

### 4. Policy 2 — “Desabilitar Defender for Servers (resource-level) por TAG”

Portal: **Policy → Definitions →** campo de busca:  
`Configure Azure Defender for Servers to be disabled for resources (resource level) with the selected tag`

![Policy disable defender for servers by tag](https://github.com/user-attachments/assets/322e4e83-efe3-471b-8841-8e5de844eb99)

- Abra a definição e clique em **Assign**.

#### Basics:
- **Scope:** selecione o mesmo Resource Group.
- **Assignment name:** por exemplo `MyRgsLabAzPol`.

![Assign policy example](https://github.com/user-attachments/assets/6cec33fb-8e0f-4ea3-8958-a93d0d1220a2)
![Assignment name example](https://github.com/user-attachments/assets/f2e22ec4-cf86-4d51-b23e-0c30a438c3ee)

#### Parameters:
- **Inclusion Tag Name:** `DefenderForServers`
- **Inclusion Tag Values:** `["Disabled"]` ← (com colchetes e aspas)
- **Effect:** `DeployIfNotExists` (padrão da built-in)

> **Opção B (UI de itens):**  
> Clique nos três pontinhos (…) → **Add value**  
> Digite `Disabled` → **OK** (vira um item da lista)

![Parameter example](https://github.com/user-attachments/assets/082dd165-5297-461f-a4e7-e4623f692452)

#### Remediation:
- Marque **Create a remediation task**.
- **Managed identity:** `System assigned managed identity`.
- **Location:** a mesma do RG (ex.: East US).

![Remediation for disable policy](https://github.com/user-attachments/assets/fbaf0ce7-d646-48f9-a40f-dc3179dc5eca)

- **Review + create → Create**

> **O que esperar**: todo recurso com a TAG `DefenderForServers=Disabled` será colocado em Free/sem plano no nível do recurso (sem cobrança de Defender for Servers para esses Arc).

![Recurso com DefenderForServers=Disabled](https://github.com/user-attachments/assets/106a7606-40c0-4cce-aa35-9d973b1934a3)

---

### 5. Execute as remediações (se não subiram automaticamente)

- Em **Policy → Assignments**, abra a assignment recém-criada.
- Aba **Remediation** → **Create remediation task**.

![Remediation aba](https://github.com/user-attachments/assets/0d32cb7b-940c-4ba8-b9e8-9bb4b8ec4a50)

- Marque **Re-evaluate compliance before remediating**.
- **Locations:** selecione a região (ex.: East US).
- **Create**.

![Remediation execution](https://github.com/user-attachments/assets/123569b7-77d7-4f73-911c-9ac37e55c6ff)

Repita para a Policy 1 (herança da TAG) se necessário.  
![Uploading image.png…]()

---

### 6. Onboard dos servidores Azure Arc

- Time de Arc: conectar as máquinas ao RG criado (ex.: `MyRgsLAbAzPols`) usando o método padrão.
- Preferencialmente, enviar também a TAG no momento do connect (se o script permitir), ou confiar na Policy 1 para herdar a TAG do RG.

> **Se não será usado MDE:**  
> Conferir que Endpoint protection/auto-provisioning no Defender for Cloud não está forçando a instalação do MDE nas máquinas Arc.  
> Caso haja MDE instalado: remover a extensão e executar offboarding no portal do Defender.

---

### 7. Validação rápida

#### 7.1 No Resource Group

- Abrir o RG → **Tags:** confirmar que o RG está com `DefenderForServers=Disabled`.

#### 7.2 Nas máquinas Azure Arc

- Azure Arc → Machines → filtrar pelo RG → abrir cada máquina:
    - Aba **Overview** ou **Tags**: conferir que a máquina tem a TAG `DefenderForServers=Disabled`.
    - Menu **Extensions**: se não quiser EDR, não deve existir `MDE.Windows/MDE.Linux`. Se existir e o cliente não quiser, remova e faça offboarding.

#### 7.3 Em Policy → Compliance

- Verifique as duas assignments do RG.
- Pode demorar alguns minutos. É normal ver o status alternando até estabilizar.

> **Observação:** Dependendo do tenant, a policy de “disable” pode aparecer como Non-compliant mesmo com o efeito aplicado (Arc em Free). Isso não impacta o faturamento. Se quiser tudo “verde”, pode usar coverage por TAG no Defender for Cloud ou Exemption.

#### 7.4 Custos

- **Cost Management → Cost analysis →** filtre por RG e por serviço “Microsoft Defender for Cloud”.
- **Esperado:** sem custo de Defender for Servers para os Arc nesse RG (após ciclo de faturamento).

---

## 8) Perguntas rápidas (FAQ)

**Precisa desligar P2 na assinatura?**  
Não. Mantemos P2 ligado na assinatura para Azure VMs. Os Arc ficam Free por TAG no resource-level.

**E se instalarem o MDE por engano?**  
Remover a extensão e offboard o dispositivo. Só remover a extensão não retira o device do portal de segurança.

**A policy mostra Non-compliant, e agora?**  
É comportamento conhecido quando a validação olha o preço no nível de assinatura. O efeito (Arc=Free) continua valendo. Se precisar “verde”, use coverage por TAG no Defender for Cloud ou Exemption.

---

## 9) Checklist final

- [ ] RG criado e com a TAG `DefenderForServers=Disabled`.
- [ ] Policy 1 (Herdar TAG) atribuída ao RG, com remediation criada.
- [ ] Policy 2 (Disable por TAG) atribuída ao RG, com remediation criada.
- [ ] Auto-provisioning do MDE desabilitado para Arc (se não quiser sensor).
- [ ] Arc conectados ao RG e com a TAG presente.
- [ ] Sem extensão MDE nas máquinas Arc (se não for usar) e offboarding feito se havia MDE.
- [ ] Compliance consultada e custos verificados após ciclo.

---


