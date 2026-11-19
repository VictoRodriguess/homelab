# 🏠 Home Lab — Infraestrutura, Redes e Cibersegurança

Home lab construído do zero em um PC reaproveitado, com **Proxmox VE**, para estudar e praticar administração Linux, Windows Server/Active Directory, redes e segurança da informação, além de hospedar serviços úteis para a casa.

> Projeto em andamento, documentado etapa por etapa, incluindo os problemas encontrados e como foram diagnosticados.

> 🔒 **Nota de segurança:** os endereços IP desta documentação foram **anonimizados** (faixa de exemplo `10.10.10.0/24`). Informações sensíveis, como IPs públicos, endereços MAC e credenciais, não são publicadas.

---

## 🎯 Objetivos

- Construir uma infraestrutura funcional, segura e documentada
- Simular um ambiente corporativo (Active Directory, GPO, SIEM)
- Praticar Blue Team e Red Team em ambiente isolado
- Hospedar serviços domésticos (DNS com bloqueio de anúncios, mídia, jogos, backup)

---

## 🖥️ Hardware

| Componente | Especificação |
|---|---|
| CPU | Intel Core i5-3330 (4 núcleos / 4 threads, VT-x e VT-d) |
| RAM | 16 GB DDR3 (2 × 8 GB, dual channel), operando a 1333 MT/s; 4 slots, suporta até 32 GB |
| Disco | HD 1 TB 7200 RPM (SATA), com LVM-thin para VMs e containers |
| Chipset | Intel Q77 (placa genérica) |
| Rede | Realtek PCIe Gigabit Ethernet (cabeada) |
| Vídeo | Intel HD 2500 integrado (GPU dedicada removida para reduzir consumo) |

Consumo estimado: 40 a 60 W em uso normal, cerca de R$ 30 a R$ 45 por mês ligado 24 horas (estimativa, sem medição na tomada).

---

## 🧱 Arquitetura

```
INTERNET
   │
Roteador ISP (10.10.10.1)
   │ cabo
Proxmox VE 9.2 — pve.home.arpa (10.10.10.10)
   └─ vmbr0 (rede doméstica)
       ├─ CT 100  adguard     10.10.10.11  DNS com filtragem
       ├─ CT 101  midia       10.10.10.12  Samba + Jellyfin
       ├─ CT 104  tailscale   10.10.10.13  VPN (subnet router)
       ├─ CT 105  romm        10.10.10.14  Biblioteca de jogos retrô
       ├─ VM DC01             10.10.10.20  AD DS + DNS (lab.home.arpa)
       └─ VM WIN11-01         10.10.10.21  Cliente do domínio

Volume LVM-thin “jogos” (50 GB) → /mnt/jogos no Proxmox
   ├─ CT 101: /jogos               (compartilhamento Samba)
   └─ CT 105: /var/lib/romm/library (biblioteca do RomM)
```

### Plano de endereçamento

| Faixa | Uso |
|---|---|
| `10.10.10.1` | Gateway (roteador) |
| `10.10.10.2 – .99` | IPs estáticos do lab |
| `10.10.10.100 – .254` | DHCP (dispositivos da casa) |

A faixa do DHCP foi reduzida no roteador para evitar conflito de IP com os serviços do lab.

---

## 📚 Etapas

| # | Etapa | Status |
|---|---|---|
| 01 | Diagnóstico de hardware e planejamento | ✅ |
| 02 | Preparação: ISO verificada por SHA-256, pendrive, BIOS (VT-x/VT-d) | ✅ |
| 03 | Instalação do Proxmox VE (UEFI, ext4/LVM-thin, IP estático) | ✅ |
| 04 | Pós-instalação e hardening inicial | ✅ |
| 05 | AdGuard Home (DNS com filtragem) | ✅ ativo no computador de testes |
| 06 | Samba + Jellyfin (compartilhamento e streaming na TV) | ✅ |
| 07 | Windows Server 2025 + Active Directory | ✅ domínio criado |
| 08 | Cliente Windows 11 no domínio + GPOs (bloqueio de tela e papel de parede) | ✅ validado com `gpresult /r` |
| 09 | Backup (regra 3-2-1) | ⏳ |
| 10 | Wazuh (SIEM) | ⏳ |
| 11 | Laboratório de pentest isolado | ⏳ |
| 12 | Acesso remoto seguro (Tailscale como subnet router) | ✅ validado pelo celular no 4G |
| 13 | Pasta de jogos compartilhada (LVM-thin + bind mount em dois containers) | ✅ |
| 14 | RomM (biblioteca de jogos e emulação no navegador) | ✅ |
| 15 | Hardening de um projetor Android da casa via ADB | ✅ |

> **AdGuard Home:** por decisão de projeto, o filtro está aplicado apenas ao computador de testes. A aplicação à rede inteira da casa foi adiada até existir um plano de contingência para quando o servidor estiver desligado.

### 🎮 Pasta de jogos e RomM

Foi criado um volume de 50 GB no LVM-thin, formatado em ext4 e montado no Proxmox em `/mnt/jogos`, com entrada no `fstab` para montar sozinho no boot. A mesma pasta foi ligada (bind mount) a dois containers: o do Samba, para acessar os arquivos pela rede, e o do RomM, que organiza a biblioteca e roda os jogos no navegador. Assim existe uma única cópia dos arquivos, com uma pasta por console (`roms/snes`, `roms/gba`, etc.).

### 📽️ Projeto paralelo: projetor Android

Um projetor Android barato da casa foi auditado via ADB pela rede. Foram removidos um app de acessibilidade suspeito e duas lojas não oficiais. Um launcher alternativo foi testado, mas o sistema do aparelho depende do launcher original para concluir o boot, então ele foi mantido. No final, a depuração ADB foi desativada, já que a porta 5555 aceitava conexões da rede sem pedir autorização.

---

## 🔐 Segurança aplicada

- Repositórios corrigidos: diagnóstico do erro `401 Unauthorized` no repositório Enterprise, troca para No-Subscription e sistema atualizado
- Usuário administrativo nominal em vez do uso diário do `root`
- Autenticação em dois fatores (TOTP) para o usuário administrativo e para o `root`, com chaves de recuperação
- Chave SSH Ed25519 configurada
- Containers **não privilegiados**; proteção contra exclusão acidental no container do RomM
- Nenhum serviço exposto à internet: sem redirecionamento de portas, acesso externo somente pelo Tailscale
- Usuário do Samba sem shell (`nologin`) e acesso restrito aos compartilhamentos
- Snapshots antes de mudanças importantes nas VMs Windows
- ADB por rede desativado no projetor depois do uso

---

## 🔍 Destaques de troubleshooting

**DNS vazando por fora do AdGuard.** O teste de bloqueio mostrava apenas ~5%. Com `ipconfig /all`, identifiquei três servidores DNS no cliente: o AdGuard, um DNS alternativo (`9.9.9.9`) e o DNS do roteador entregue via IPv6. O Windows distribuía as consultas entre eles. Deixando apenas o AdGuard, o bloqueio subiu para ~70%. Reintroduzindo o DNS alternativo, caiu para ~8%, confirmando a causa.

**Container sem rede.** `apt update` falhava com `Temporary failure resolving`. O `ping` mostrou `Destination Host Unreachable` com origem em um IP com um octeto digitado errado (equivalente a `10.11.10.12` em vez de `10.10.10.12`). Após a correção, a rede voltou.

**Kerberos e horário.** O fuso horário do Windows Server foi corrigido antes da criação do domínio, já que o Kerberos recusa autenticação com diferença de relógio acima de 5 minutos.

**Montagem que sumia no reboot.** O volume montado com `mount` não voltava depois de reiniciar. A solução foi uma entrada no `/etc/fstab` com a opção `nofail`, para o servidor ligar mesmo se o volume falhar.

**Permissões em container não privilegiado.** Dentro do container, o usuário tinha UID 1000, mas no host ele aparece como 101000, porque o Proxmox soma 100000 aos IDs. A pasta compartilhada recebeu `chown 101000:101000` no host para o Samba conseguir gravar.

**Emulador com “core undefined”.** O RomM mostrava `Error downloading core (undefined-legacy-wasm.data)`. Os núcleos estavam instalados (confirmado com `find`); o problema era o jogo cadastrado com zero arquivos vinculados. “Atualizar arquivos” e um novo scan resolveram.

**Fonte de metadados recusada.** O ScreenScraper retornava “chave de API inválida” mesmo com usuário e senha corretos. Pela documentação do RomM, instalações fora da imagem Docker oficial não trazem as credenciais de desenvolvedor exigidas pelo serviço.

**Projetor preso na tela de boot.** Ao desativar o launcher original, o aparelho não terminava de iniciar. Como o ADB continuava acessível durante o boot, foi possível reverter remotamente. Lição extra: `pm enable` respondia `enabled`, mas só passou a valer com `--user 0`, o mesmo usuário usado no `disable-user`.

---

## 🧠 Aprendizados

- Validar cada etapa com um teste (`gpresult`, `df -h`, `ls`, teste de bloqueio) em vez de confiar só no comando executado
- Fazer uma mudança por vez e saber de antemão como reverter
- Manter um caminho de acesso de emergência (ADB, Shell do Proxmox, snapshot) antes de mexer em algo que pode impedir o boot
- Containers não privilegiados exigem atenção ao mapeamento de UID/GID
- Ler a documentação oficial quando o erro não faz sentido: a causa costuma estar no modo de instalação
- Documentar enquanto faz, incluindo os erros

---

## 🧭 Próximos passos

**Pendências de segurança**

- Desativar login SSH por senha (somente chave)
- Tailscale: desativar a expiração da chave do servidor e ativar a aprovação de novos dispositivos
- Avaliar o AdGuard para toda a rede, com plano de contingência
- Backup automatizado em HD externo

**Roteiro de evolução**

| Fase | Foco | O que entra |
|---|---|---|
| 1 | Turbinar o servidor | RAM para 32 GB, SSD para sistema e VMs, nobreak |
| 2 | Rede de empresa | Switch gerenciável com VLANs, firewall OPNsense, Wi-Fi com redes separadas |
| 3 | Casa inteligente | Home Assistant com Zigbee, sensores e tomadas medidoras |
| 4 | Cluster e NAS | Segundo nó Proxmox, HDs espelhados, Proxmox Backup Server |

Em paralelo: Wazuh + Sysmon coletando logs do Linux e do Windows, e laboratório ofensivo em rede isolada (Kali × alvos vulneráveis).

---

## 🛠️ Tecnologias

Proxmox VE · Debian 13 · LXC · KVM/QEMU · LVM-thin · AdGuard Home · Samba · Jellyfin · Tailscale · RomM · EmulatorJS · Windows Server 2025 · Active Directory · GPO · DNS · PowerShell · Bash · SSH · TOTP · ADB
