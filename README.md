Objetivo

Manter Azure VMs sob Defender for Servers Plan 2 (P2) na assinatura, enquanto servidores Azure Arc (e qualquer outro recurso marcado) ficam em Free/sem plano no nível do recurso, usando TAG + Azure Policy (com remediação).
Tudo que “subir” no RG rg-azurearc-... não recebe Defender for Servers (nem P1 nem P2) → fica Free/sem plano.


 
Passo 0 — Entender o mecanismo (oficial)

O Defender for Servers pode ser habilitado/desabilitado por recurso (VM/Arc/VMSS), além do nível de assinatura. A doc afirma: “Embora recomendemos habilitar por assinatura, você pode habilitar no nível do recurso (Azure VMs e Azure Arc-enabled servers).” 
[Microsoft Learn](https://learn.microsoft.com/en-us/azure/defender-for-cloud/tutorial-enable-servers-plan)

Há uma policy built-in que desabilita o Defender for Servers para recursos que tenham uma tag específica (vale para VM, VMSS e Arc Machines). 
https://learn.microsoft.com/en-us/azure/azure-arc/servers/policy-reference

Para garantir que todos os recursos do RG recebam a tag automaticamente, use a policy built-in “Inherit a tag from the resource group (if missing)”. 
https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies?

Passo 1 — Coloque a tag no Resource Group

No RG rg-azurearc-...:

Tags → Add tag

Key: DefenderForServers

Value: Disabled 
Salve.

<img width="1221" height="575" alt="image" src="https://github.com/user-attachments/assets/6c7d2893-6fec-4e45-b10b-c70f988cbf62" />

(Depois vamos herdar essa tag para todo recurso criado no RG.)
Docs de políticas de tag: https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies?
Se vc não quer sensor MDE nos Arc: combine com o time responsável para offboarding do MDE e remover a extensão MDE nas máquinas Arc.


1) Crie/eleja o Resource Group para os servidores Arc

No portal, Resource groups → Create (se já existir, pule).

Campos:

Resource group: MyRgsLabAzPolArc (ou o nome do seu RG)

Region: a mesma que você vai usar nos Arc (ex.: East US)

Review + create → Create.

2) Coloque a TAG no Resource Group (para herdar nos recursos)

Abra o seu RG (ex.: rg-azurearc-itsbx-us).

Menu lateral Tags → Add:

Name: DefenderForServers

Value: Disabled

Save.

Dica: essa TAG no RG será herdada pelos recursos com a policy do próximo passo.

3) Policy 1 — “Herdar TAG do RG se estiver faltando”

Portal: Policy → Definitions → campo de busca:
Inherit a tag from the resource group if missing

Abra a definição e clique em Assign.

Basics:

Scope: selecione o Resource Group onde estão/estarão os Arc.

Assignment name: por exemplo RG-ARC-InheritTag-DefenderForServers.


<img width="992" height="710" alt="image" src="https://github.com/user-attachments/assets/3fc81f27-673c-49a5-982c-6323dcdfe958" />


Parameters:

Tag Name: DefenderForServers

<img width="777" height="297" alt="image" src="https://github.com/user-attachments/assets/861c8bc3-b675-4e04-93ba-69bdab8274fe" />


Remediation:

Marque Create a remediation task.

Managed identity: deixe System assigned managed identity.

Location: a mesma do RG (ex.: East US).
<img width="759" height="702" alt="image" src="https://github.com/user-attachments/assets/9af76b4f-9954-4a14-b8fd-614cd921566c" />


Review + create → Create.

O que esperar: recursos novos e existentes do RG receberão a TAG do RG (caso não tenham).

<img width="1669" height="917" alt="image" src="https://github.com/user-attachments/assets/cc0ba65a-a446-487f-9d75-7b51b5c2effd" />

4) Policy 2 — “Desabilitar Defender for Servers (resource-level) por TAG”
Portal: Policy → Definitions → campo de busca:
Configure Azure Defender for Servers to be disabled for resources (resource level) with the selected tag
<img width="1656" height="415" alt="image" src="https://github.com/user-attachments/assets/322e4e83-efe3-471b-8841-8e5de844eb99" />

Abra a definição e clique em Assign.

Basics:

Scope: selecione o mesmo Resource Group.

Assignment name: por exemplo MyRgsLabAzPol.
<img width="968" height="268" alt="image" src="https://github.com/user-attachments/assets/6cec33fb-8e0f-4ea3-8958-a93d0d1220a2" />
<img width="968" height="709" alt="image" src="https://github.com/user-attachments/assets/f2e22ec4-cf86-4d51-b23e-0c30a438c3ee" />


Parameters:

Inclusion Tag Name: DefenderForServers

Inclusion Tag Values: ["Disabled"] ← (com colchetes e aspas)

Effect: DeployIfNotExists (padrão da built-in)

Opção B (UI de itens)

Clique nos três pontinhos (…) → Add value

Digite Disabled → OK (vira um item da lista)

Effect: DeployIfNotExists

Effect: deixe o padrão que a tela mostrar (normalmente já vem correto).
<img width="912" height="409" alt="image" src="https://github.com/user-attachments/assets/082dd165-5297-461f-a4e7-e4623f692452" />


Remediation:

Marque Create a remediation task.

Managed identity: System assigned managed identity.

Location: a mesma do RG (ex.: East US).
<img width="1027" height="888" alt="image" src="https://github.com/user-attachments/assets/fbaf0ce7-d646-48f9-a40f-dc3179dc5eca" />

Review + create → Create.

O que esperar: todo recurso com a TAG DefenderForServers=Disabled será colocado em Free/sem plano no nível do recurso (sem cobrança de Defender for Servers para esses Arc).


<img width="468" height="505" alt="image" src="https://github.com/user-attachments/assets/106a7606-40c0-4cce-aa35-9d973b1934a3" />


5) Execute as remediações (se não subiram automaticamente)

Em Policy → Assignments, abra a assignment recém-criada.

Aba Remediation → Create remediation task. 
<img width="1652" height="632" alt="image" src="https://github.com/user-attachments/assets/0d32cb7b-940c-4ba8-b9e8-9bb4b8ec4a50" />


Marque Re-evaluate compliance before remediating.

Locations: selecione a região (ex.: East US).

Create.
<img width="1667" height="884" alt="image" src="https://github.com/user-attachments/assets/123569b7-77d7-4f73-911c-9ac37e55c6ff" />


Repita para a Policy 1 (herança da TAG) se necessário.
![Uploading image.png…]()


6) Onboard dos servidores Azure Arc

Time de Arc: conectar as máquinas ao RG criado (ex.: MyRgsLAbAzPols) usando o método padrão.

Preferencialmente, enviar também a TAG no momento do connect (se o script permitir), ou confiar na Policy 1 para herdar a TAG do RG.

Se não será usado MDE:

Conferir que Endpoint protection/auto-provisioning no Defender for Cloud não está forçando a instalação do MDE nas máquinas Arc.

Caso haja MDE instalado: remover a extensão e executar offboarding no portal do Defender.

7) Validação rápida
7.1 No Resource Group

Abrir o RG → Tags: confirmar que o RG está com DefenderForServers=Disabled.

7.2 Nas máquinas Azure Arc

Azure Arc → Machines → filtrar pelo RG → abrir cada máquina:

Aba Overview ou Tags: conferir que a máquina tem a TAG DefenderForServers=Disabled.

Menu Extensions: se não quiser EDR, não deve existir MDE.Windows/MDE.Linux. Se existir e o cliente não quiser, remova e faça offboarding.

7.3 Em Policy → Compliance

Verifique as duas assignments do RG.

Pode demorar alguns minutos. É normal ver o status alternando até estabilizar.

Observação: dependendo do tenant, a policy de “disable” pode aparecer como Non-compliant mesmo com o efeito aplicado (Arc em Free). Isso não impacta o faturamento. Se quiser tudo “verde”, peça ao time de segurança para usar Manage/Customize coverage (exclusão por TAG) ou criar Exemption.

7.4 Custos

Cost Management → Cost analysis → filtre por RG e por serviço “Microsoft Defender for Cloud”.

Esperado: sem custo de Defender for Servers para os Arc nesse RG (após ciclo de faturamento).

8) Perguntas rápidas (FAQ)

Precisa desligar P2 na assinatura?
Não. Mantemos P2 ligado na assinatura para Azure VMs. Os Arc ficam Free por TAG no resource-level.

E se instalarem o MDE por engano?
Remover a extensão e offboard o dispositivo. Só remover a extensão não retira o device do portal de segurança.

A policy mostra Non-compliant, e agora?
É comportamento conhecido quando a validação olha o preço no nível de assinatura. O efeito (Arc=Free) continua valendo. Se precisar “verde”, use coverage por TAG no Defender for Cloud ou Exemption.

9) Checklist final

 RG criado e com a TAG DefenderForServers=Disabled.

 Policy 1 (Herdar TAG) atribuída ao RG, com remediation criada.

 Policy 2 (Disable por TAG) atribuída ao RG, com remediation criada.

 Auto-provisioning do MDE desabilitado para Arc (se não quiser sensor).

 Arc conectados ao RG e com a TAG presente.

 Sem extensão MDE nas máquinas Arc (se não for usar) e offboarding feito se havia MDE.

 Compliance consultada e custos verificados após ciclo.



