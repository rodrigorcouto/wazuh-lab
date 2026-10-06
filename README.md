# Relatório Técnico de Implantação e Homologação do Wazuh On Premises

## Implementação, troubleshooting, integração Microsoft Defender, Sysmon e configuração centralizada

| Campo | Informação |
|---|---|
| Data | 06 de outubro de 2026 |
| Responsável técnico | Equipe de Segurança da Informação |
| Cliente/Laboratório | Laboratório de Cibersegurança |
| Versão do documento | 1.0 |
| Classificação | Uso corporativo e homologação interna |

---

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Preparação do ambiente](#2-preparação-do-ambiente)
3. [Troubleshooting de armazenamento](#3-troubleshooting-de-armazenamento)
4. [Correção da GPT](#4-correção-da-gpt)
5. [Expansão LVM](#5-expansão-lvm)
6. [Instalação do Wazuh](#6-instalação-do-wazuh)
7. [Troubleshooting da instalação](#7-troubleshooting-da-instalação)
8. [Remoção forçada do pacote](#8-remoção-forçada-do-pacote)
9. [Instalação bem-sucedida](#9-instalação-bem-sucedida)
10. [Validação dos serviços](#10-validação-dos-serviços)
11. [Primeiro acesso](#11-primeiro-acesso)
12. [Implantação do agente Windows](#12-implantação-do-agente-windows)
13. [Testes de homologação Windows](#13-testes-de-homologação-windows)
14. [Integração Microsoft Defender](#14-integração-microsoft-defender)
15. [Ajuste no `ossec.conf`](#15-ajuste-no-ossecconf)
16. [Teste EICAR](#16-teste-eicar)
17. [Resultados do teste EICAR](#17-resultados-do-teste-eicar)
18. [Instalação do Sysmon](#18-instalação-do-sysmon)
19. [Integração do Sysmon](#19-integração-do-sysmon)
20. [Centralização de configurações](#20-centralização-de-configurações)
21. [Agent Groups](#21-agent-groups)
22. [Troubleshooting de grupos](#22-troubleshooting-de-grupos)
23. [Validação da configuração centralizada](#23-validação-da-configuração-centralizada)
24. [Boas práticas corporativas](#24-boas-práticas-corporativas)
25. [Conclusão](#25-conclusão)

---

## 1. Visão geral

Este relatório documenta a implantação laboratorial de uma plataforma **Wazuh On Premises**, incluindo os desvios identificados, as correções aplicadas, as integrações com Microsoft Defender e Sysmon e a homologação de um agente Windows. O objetivo é oferecer rastreabilidade para auditoria, repetição do procedimento e evolução segura para produção.

O Wazuh é uma plataforma open source de segurança para centralização de logs, detecção baseada em regras, monitoramento de integridade, inventário e avaliação de vulnerabilidades. Em uma arquitetura SIEM, converte telemetria bruta em alertas passíveis de investigação pelo SOC.

| Componente | Papel operacional |
|---|---|
| Wazuh Manager | Recebe eventos, decodifica logs, aplica regras e gerencia agentes. |
| Wazuh Indexer | Armazena e indexa documentos de alerta usando OpenSearch. |
| Filebeat | Encaminha os alertas locais do Manager ao Indexer. |
| Wazuh Dashboard | Fornece interface de investigação, painéis e administração. |
| Agentes Wazuh | Coletam telemetria dos endpoints e aplicam políticas compartilhadas. |

```text
Windows Agent
     |
Wazuh Manager
     |
Filebeat
     |
Wazuh Indexer (OpenSearch)
     |
Wazuh Dashboard
```

## 2. Preparação do ambiente

O servidor foi preparado com **Ubuntu Server 22.04 LTS**. A escolha considera maturidade do ciclo LTS, ampla compatibilidade com os componentes Wazuh e disponibilidade de atualizações de segurança.

| Recurso | Capacidade inicial | Avaliação para laboratório | Risco se mantido |
|---|---:|---|---|
| vCPU | 11 | Suficiente para baixa simultaneidade | Atrasos de indexação sob carga |
| Memória | 8 GB | Mínimo operacional para todos os serviços | Pressão de heap e uso de swap |
| Disco | 24 GB | Insuficiente para retenção contínua | Falha por esgotamento de espaço |

A configuração inicial restrita é aceitável para laboratório, porém não representa dimensionamento de produção. O cálculo corporativo deve considerar EPS, quantidade de agentes, fontes de dados, prazo de retenção, réplicas, janelas de busca e requisitos de continuidade.

## 3. Troubleshooting de armazenamento

O disco virtual foi expandido para **100 GB** no Hyper-V. O Linux, porém, não passou a utilizar imediatamente a capacidade adicional porque a tabela GPT de backup permaneceu no limite antigo do dispositivo.

```bash
lsblk
fdisk -l
```

As mensagens identificadas foram:

```text
GPT PMBR size mismatch
The backup GPT table is not on the end of the device
```

O PMBR (*Protective MBR*) protege discos GPT contra ferramentas legadas. A divergência de tamanho mostra que sua geometria registrada ainda corresponde ao disco anterior. A GPT possui estruturas primárias no início e estruturas de backup no final da unidade; após a expansão, a cópia de backup permaneceu no antigo final lógico.

## 4. Correção da GPT

```bash
sudo parted /dev/sda
```

No GNU Parted, foi selecionada a opção **Fix**. A ação reposiciona os metadados GPT de backup no novo final do dispositivo, sem recriar partições ou formatar volumes.

```text
+--------------------------- /dev/sda 100 GB ---------------------------+
| GPT primária | partições existentes | espaço ampliado | GPT backup      |
+------------------------------------------------------------------------+

Antes: GPT backup no antigo limite de 24 GB
Depois: GPT backup no novo limite físico do disco
```

> **Observação:** em ambientes com múltiplos discos, valide rigorosamente o dispositivo-alvo antes de aplicar alterações de particionamento. Em produção, execute o procedimento em janela de mudança e com backup ou snapshot consistente.

## 5. Expansão LVM

Após corrigir a GPT, a expansão foi aplicada nas camadas de armazenamento LVM.

```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

```text
Disco /dev/sda
  -> Partição /dev/sda3
      -> Physical Volume
          -> Volume Group ubuntu-vg
              -> Logical Volume ubuntu-lv
                  -> Filesystem ext4
```

| Camada | Função | Comando |
|---|---|---|
| GPT e partição | Amplia a partição para a área disponível | `growpart /dev/sda 3` |
| Physical Volume | Atualiza a capacidade conhecida pelo LVM | `pvresize /dev/sda3` |
| Logical Volume | Entrega extents livres ao volume lógico | `lvextend -l +100%FREE ...` |
| Filesystem | Expande o ext4 para utilizar o LV | `resize2fs ...` |

Após a expansão, recomenda-se validar com `lsblk`, `pvs`, `vgs`, `lvs` e `df -h`.

## 6. Instalação do Wazuh

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

A opção `-a` executa a implantação all-in-one de laboratório, instalando o Wazuh Manager, Wazuh Indexer, Filebeat e Wazuh Dashboard. O instalador também prepara certificados e credenciais de comunicação entre os componentes.

| Componente | Função |
|---|---|
| Wazuh Manager | Análise de eventos, regras, decoders, registro e gestão de agentes. |
| Wazuh Indexer | Persistência e pesquisa dos eventos de segurança. |
| Filebeat | Publicação de alertas do Manager para o Indexer. |
| Wazuh Dashboard | Visualização, investigação e administração da solução. |

O modo all-in-one é apropriado para homologação. Em produção, a arquitetura deve ser dimensionada e distribuída de acordo com volume, criticidade e requisitos de alta disponibilidade.

## 7. Troubleshooting da instalação

Durante a instalação foi identificado o erro:

```text
Wazuh manager already installed
```

O diagnóstico foi realizado com:

```bash
dpkg -l | grep wazuh
```

Esse resultado indica presença anterior do pacote, mesmo que a instalação esteja incompleta ou que o serviço não esteja funcional. Em distribuições Debian/Ubuntu, os scripts de manutenção controlam fases do ciclo do pacote.

| Script | Momento de execução | Impacto de falha |
|---|---|---|
| `prerm` | Antes de remover ou atualizar | Pode interromper a desinstalação. |
| `postrm` | Após a remoção de arquivos | Pode impedir a conclusão e a limpeza de estado. |
| `postinst` | Após a instalação | Serviço ou configuração podem ficar incompletos. |

## 8. Remoção forçada do pacote

Após preservar os scripts problemáticos e renomeá-los para impedir sua execução, foi utilizada a remoção corretiva:

```bash
dpkg --remove --force-remove-reinstreq wazuh-manager
```

`--force-remove-reinstreq` permite remover um pacote marcado como requerendo reinstalação. É uma medida excepcional: pode deixar diretórios, contas de serviço, arquivos de configuração, chaves e filas residuais.

| Risco | Mitigação |
|---|---|
| Perda de configuração | Preservar cópia dos arquivos antes da remoção. |
| Resíduos de instalação | Revisar `/var/ossec`, unidades systemd e repositórios. |
| Reinstalação inconsistente | Reinstalar apenas após validar a limpeza. |
| Desvio de segurança | Regenerar e validar certificados e credenciais. |

## 9. Instalação bem-sucedida

Após a remoção do estado inconsistente, a instalação foi repetida com sucesso. A validação deve confirmar a cadeia ponta a ponta e não apenas a finalização do instalador.

```text
Evento no endpoint
  -> agente Wazuh
  -> Wazuh Manager
  -> alerts.json
  -> Filebeat
  -> Wazuh Indexer
  -> Wazuh Dashboard
```

Pontos de falha comuns incluem conectividade, chave do agente, decoder ou regra, serviço Filebeat, certificado, saúde do Indexer e filtros temporais do Dashboard.

## 10. Validação dos serviços

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
```

O estado `active (running)` confirma que o processo principal está em execução sob supervisão do systemd. Isso é necessário, mas não suficiente: o serviço pode permanecer ativo mesmo com falhas de conectividade, certificados, espaço em disco ou indexação.

| Serviço | Validação complementar |
|---|---|
| `wazuh-manager` | Registro de agentes, logs de análise e fila de eventos. |
| `wazuh-indexer` | Saúde do cluster, shards e utilização de disco. |
| `wazuh-dashboard` | Acesso HTTPS, autenticação e consulta de alertas. |

## 11. Primeiro acesso

O acesso inicial ao Dashboard foi realizado em:

```text
https://192.168.1.28
```

O alerta apresentado pelo navegador é esperado quando o serviço utiliza certificado autoassinado. Em laboratório, a aceitação controlada é adequada. Para produção, recomenda-se certificado emitido por PKI corporativa ou autoridade confiável, associado a um FQDN corporativo e com rotação definida.

| Parâmetro | Laboratório | Produção recomendada |
|---|---|---|
| Endereço | `https://192.168.1.28` | `https://wazuh.empresa.example` |
| Certificado | Autoassinado | PKI corporativa ou CA confiável |
| Acesso | Administrativo de homologação | RBAC, MFA e rede restrita |

## 12. Implantação do agente Windows

O endpoint **Laboratorio-PC** foi integrado como agente de ID **001**. O registro estabelece uma identidade criptográfica que permite ao Manager autenticar a telemetria. A chave de enrollment é exclusiva do agente e deve ser tratada como informação sensível.

| Atributo | Valor homologado | Verificação |
|---|---|---|
| Hostname | `Laboratorio-PC` | Identificação no Dashboard |
| ID | `001` | Registro no Manager |
| Plataforma | Windows | Canal de eventos e serviço do agente |
| Estado | Active | Heartbeat e comunicação válidos |

## 13. Testes de homologação Windows

```bat
net user wazuhteste Senha@123 /add
net user wazuhteste /delete
```

| Cenário | Ação | Resultado esperado | Regra |
|---|---|---|---|
| Criação de usuário | `net user ... /add` | Alerta de criação de conta local | `60109` |
| Exclusão de usuário | `net user ... /delete` | Alerta de remoção de conta local | `60110` |
| Logon | Autenticação válida controlada | Evento de logon correlacionável | Conforme regra aplicável |
| Falha de logon | Senha inválida em teste | Evento de autenticação recusada | Conforme regra aplicável |

Os testes devem ser realizados com conta de laboratório, horário registrado e validação dos campos de usuário, origem, hostname, Event ID e nível de regra.

## 14. Integração Microsoft Defender

O Microsoft Defender gerava eventos locais, mas eles não eram inicialmente enviados ao Wazuh. A causa foi a ausência de coleta do canal:

```text
Microsoft-Windows-Windows Defender/Operational
```

O Wazuh não coleta automaticamente todos os canais de eventos Windows. A fonte deve estar explicitamente configurada e ser validada localmente antes da análise no SIEM.

| Camada | Pergunta de diagnóstico |
|---|---|
| Defender local | O evento existe no canal Operational? |
| Windows Event Log | O canal está habilitado e acessível ao agente? |
| `ossec.conf` | Há `localfile` com formato correto? |
| Agente Wazuh | O serviço recarregou sem erro? |
| Manager e Dashboard | O evento chegou e foi correlacionado? |

## 15. Ajuste no `ossec.conf`

```xml
<localfile>
  <location>Microsoft-Windows-Windows Defender/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

O formato `eventchannel` informa ao Wazuh que a fonte é um canal estruturado do Windows Event Log, preservando atributos necessários para regras e investigação. Depois da alteração, o agente deve ser reiniciado ou recarregado, e a chegada de eventos deve ser validada ponta a ponta.

Para escala corporativa, essa definição deve migrar para configuração compartilhada de grupos, em vez de permanecer como alteração manual de endpoint.

## 16. Teste EICAR

O EICAR é uma cadeia de teste padronizada e inofensiva, utilizada para validar soluções antimalware sem introduzir malware real.

```text
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

| Benefício | Evidência produzida |
|---|---|
| Baixo risco | Não utiliza código malicioso ativo. |
| Repetibilidade | Pode ser executado com operador e horário controlados. |
| Rastreabilidade | Gera eventos operacionais do Defender. |
| Validação ponta a ponta | Confirma ingestão e regras no Wazuh. |

O teste comprova o fluxo Defender -> Windows Event Log -> Agente Wazuh -> Manager -> Indexer -> Dashboard. Ele não substitui testes de resposta a incidentes nem avaliação de técnicas adversárias reais.

## 17. Resultados do teste EICAR

| Event ID | Significado operacional | Regra Wazuh | Uso no SOC |
|---:|---|---:|---|
| 1116 | Detecção de ameaça pelo Microsoft Defender | 62123 | Gatilho de triagem e investigação. |
| 1117 | Ação de remediação aplicada pelo Defender | 62124 | Confirmação de contenção ou limpeza. |

Uma detecção sem ação de remediação deve ser investigada. Preservar, quando disponível, hash, caminho, usuário, processo associado e estado final do artefato melhora a qualidade de resposta.

## 18. Instalação do Sysmon

```powershell
winget install Microsoft.Sysinternals.Sysmon
Get-Service *sysmon*
Get-WinEvent -ListLog *Sysmon*
```

O Sysmon amplia a visibilidade de endpoint por meio de eventos detalhados de criação de processo, conexões de rede, criação de arquivos, alterações de registro e outras atividades relevantes. Pode registrar linha de comando, hash, processo pai e GUIDs de processo, atributos fundamentais para análise forense.

| Validação | Objetivo |
|---|---|
| `Get-Service *sysmon*` | Confirmar instalação e execução do serviço. |
| `Get-WinEvent -ListLog *Sysmon*` | Confirmar o canal Operational. |
| Event Viewer | Verificar eventos estruturados e retenção. |

Em produção, o Sysmon deve utilizar uma configuração XML versionada, aprovada e testada em piloto. Uma configuração inadequada pode gerar volume excessivo ou lacunas de telemetria.

## 19. Integração do Sysmon

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

| Event ID | Telemetria | Exemplo de uso MITRE ATT&CK |
|---:|---|---|
| 1 | Criação de processo | T1059: análise de processo pai e linha de comando. |
| 3 | Conexão de rede | T1071: conexões inesperadas ou C2. |
| 11 | Criação de arquivo | T1105: artefatos em diretórios sensíveis. |

Os eventos devem ser correlacionados com Defender, autenticação Windows, DNS, proxy e firewall. Uma cadeia de processo suspeito, conexão externa e escrita de arquivo possui valor analítico maior que cada evento isolado.

## 20. Centralização de configurações

| Escala | Método manual | Configuração centralizada |
|---|---|---|
| 1 máquina | Viável para homologação inicial | Útil para validar política antes da distribuição. |
| 100 máquinas | Alto risco de divergência | Distribuição por grupo e controle de mudança. |
| 1000 máquinas | Operação impraticável | Política versionada, segmentada e monitorada. |

O Wazuh permite associar agentes a grupos e distribuir um `agent.conf` compartilhado. A política deve ser tratada como código: versionada, revisada, testada em grupo piloto, liberada gradualmente e com possibilidade de reversão.

## 21. Agent Groups

Foi definido o grupo **Windows** para concentrar a política comum dos endpoints Windows.

```text
/var/ossec/etc/shared/Windows/agent.conf
```

| Fonte | Objetivo |
|---|---|
| Microsoft Defender Operational | Detecções e ações de proteção do endpoint. |
| PowerShell Operational | Execução administrativa, módulos e script blocks. |
| Sysmon Operational | Processos, rede, arquivos e telemetria avançada. |

Em ambiente corporativo, recomenda-se segmentar grupos por função e criticidade, por exemplo: `Windows Workstations`, `Windows Servers`, `Domain Controllers` e `High Value Assets`.

## 22. Troubleshooting de grupos

Linux diferencia maiúsculas e minúsculas. Portanto, `Windows` e `windows` são nomes distintos. Uma divergência de capitalização pode fazer com que a configuração compartilhada não seja encontrada ou não seja aplicada ao agente esperado.

| Item | Correto | Possível falha |
|---|---|---|
| Nome do grupo | `Windows` | `windows` |
| Diretório | `/var/ossec/etc/shared/Windows` | `/var/ossec/etc/shared/windows` |
| Resultado | Política encontrada e aplicada | Política ausente ou não associada |

O diagnóstico deve confirmar o nome no Manager, a associação efetiva do agente, a existência do diretório e os logs do Manager e do endpoint após a distribuição.

## 23. Validação da configuração centralizada

```text
Agent is reloading due to shared configuration changes
```

Essa mensagem comprova que o agente detectou alteração de configuração compartilhada distribuída pelo Manager e iniciou uma recarga controlada. A evidência demonstra a sequência: política atualizada no servidor, entrega ao agente e aplicação no endpoint sem edição manual local.

| Etapa | Evidência esperada |
|---|---|
| Alteração no `agent.conf` | Controle de versão e revisão de mudança. |
| Distribuição | Agente recebe a configuração compartilhada. |
| Recarga | Log com mensagem de reload. |
| Teste funcional | Evento do canal chega ao Dashboard. |
| Monitoramento | Ausência de erros persistentes ou saturação de fila. |

## 24. Boas práticas corporativas

| Domínio | Recomendação prática |
|---|---|
| Sysmon | Aplicar XML versionado, baseado em casos de uso MITRE e validado em piloto. |
| Microsoft Defender | Centralizar políticas, enviar telemetria e revisar exclusões. |
| PowerShell Operational | Habilitar Script Block Logging, Module Logging e transcrição conforme política. |
| Active Directory | Coletar eventos de DCs, grupos privilegiados e autenticação anômala. |
| GPO e Intune | Distribuir auditoria e agentes com anéis de implantação. |
| SCCM | Integrar inventário, atualização e estado de endpoint quando aplicável. |
| Microsoft Sentinel | Definir integração por caso de uso, responsabilidade e contrato de dados. |
| CrowdStrike | Correlacionar alertas de endpoint e estabelecer fluxo de tratamento. |
| Fortigate | Ingerir logs de segurança, VPN, tráfego e eventos de bloqueio. |
| Microsoft 365 | Integrar auditoria, identidade, e-mail e eventos de colaboração. |

Além de integrações, a operação deve abranger RBAC, MFA, backup testado, retenção com ILM, monitoração de agentes inativos, gestão de certificados, controle de capacidade e revisão periódica de regras ruidosas.

## 25. Conclusão

A implantação laboratorial do Wazuh On Premises foi concluída com sucesso após a correção da expansão de armazenamento e a remoção do estado residual do pacote. A solução demonstrou operação do Manager, Indexer, Filebeat e Dashboard, além de comunicação válida com o agente Windows.

A homologação confirmou coleta de eventos de conta local, integração com Microsoft Defender, detecção EICAR pelos eventos 1116 e 1117, preparação da telemetria Sysmon e atualização de configuração por grupos de agentes. Antes da produção, devem ser formalizados dimensionamento, arquitetura de disponibilidade, certificados confiáveis, governança de acesso, retenção, backup e casos de uso do SOC.

### Checklist de homologação

- [x] Wazuh instalado
- [x] Indexer operacional
- [x] Dashboard operacional
- [x] Agente operacional
- [x] Defender integrado
- [x] Sysmon integrado
- [x] Configuração centralizada validada
- [x] EICAR detectado
- [x] Laboratório homologado
