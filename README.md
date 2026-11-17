# 🏠 Home Lab — Infraestrutura, Redes e Cibersegurança

Home lab construído do zero em um PC reaproveitado, com **Proxmox VE**, para estudar e praticar administração Linux, Windows Server/Active Directory, redes e segurança da informação, além de hospedar serviços úteis para a casa.

> Projeto em andamento, documentado etapa por etapa, incluindo os problemas encontrados e como foram diagnosticados.

> 🔒 **Nota de segurança:** os endereços IP desta documentação foram **anonimizados** (faixa de exemplo `10.10.10.0/24`). Informações sensíveis, como IPs públicos, endereços MAC e credenciais, não são publicadas.

---

## 🎯 Objetivos

- Construir uma infraestrutura funcional, segura e documentada
- Simular um ambiente corporativo (Active Directory, GPO, SIEM)
- Praticar Blue Team e Red Team em ambiente isolado
- Hospedar serviços domésticos (DNS com bloqueio de anúncios, mídia, backup)

---

## 🖥️ Hardware

| Componente | Especificação |
|---|---|
| CPU | Intel Core i5-3330 (4 núcleos / 4 threads, VT-x e VT-d) |
| RAM | 16 GB DDR3 1600 (2 × 8 GB, dual channel; placa suporta até 32 GB) |
| Disco | HD 1 TB 7200 RPM (SATA) |
| Chipset | Intel Q77 (placa genérica) |
| Rede | Realtek PCIe Gigabit Ethernet (cabeada) |
| Vídeo | Intel HD 2500 integrado (GPU dedicada removida para reduzir consumo) |

---

## 🧱 Arquitetura

```
                 INTERNET
                    │
           ┌────────┴────────┐
           │  Roteador ISP   │  10.10.10.1
           └────────┬────────┘
                    │ cabo
   ┌────────────────┴──────────────────────────────┐
   │  Proxmox VE 9.2 (pve.home.arpa)  10.10.10.10  │
   │                                               │
   │  vmbr0 (rede doméstica)                       │
   │   ├─ CT 100  adguard   10.10.10.11  DNS       │
   │   ├─ CT 101  midia     10.10.10.12  Samba +   │
   │   │                                 Jellyfin  │
   │   ├─ VM DC01           10.10.10.20  AD DS/DNS │
   │   └─ VM WIN11-01       (em instalação)        │
   └───────────────────────────────────────────────┘
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
| 08 | Cliente Windows 11 no domínio + GPOs | 🔄 |
| 09 | Backup (regra 3-2-1) | ⏳ |
| 10 | Wazuh (SIEM) | ⏳ |
| 11 | Laboratório de pentest isolado | ⏳ |
| 12 | Acesso remoto seguro (Tailscale) | ⏳ |

> **AdGuard Home:** por decisão de projeto, o filtro está aplicado apenas ao computador de testes. A aplicação à rede inteira da casa foi adiada até existir um plano de contingência para quando o servidor estiver desligado.

---

## 🔐 Segurança aplicada

- Repositórios corrigidos: diagnóstico do erro `401 Unauthorized` no repositório Enterprise, troca para No-Subscription e sistema atualizado
- Usuário administrativo nominal em vez do uso diário do `root`
- Autenticação em dois fatores (TOTP) para o usuário administrativo e para o `root`, com chaves de recuperação
- Chave SSH Ed25519 configurada
- Containers **não privilegiados**
- Nenhum serviço exposto à internet (sem redirecionamento de portas; UPnP desativado no Jellyfin)
- Usuário do Samba sem shell (`nologin`) e acesso restrito ao compartilhamento

---

## 🔍 Destaques de troubleshooting

**DNS vazando por fora do AdGuard.** O teste de bloqueio mostrava apenas ~5%. Com `ipconfig /all`, identifiquei três servidores DNS no cliente: o AdGuard, um DNS alternativo (`9.9.9.9`) e o DNS do roteador entregue via IPv6. O Windows distribuía as consultas entre eles. Deixando apenas o AdGuard, o bloqueio subiu para ~70%. Reintroduzindo o DNS alternativo, caiu para ~8%, confirmando a causa.

**Container sem rede.** `apt update` falhava com `Temporary failure resolving`. O `ping` mostrou `Destination Host Unreachable` com origem em um IP com um octeto digitado errado (equivalente a `10.11.10.12` em vez de `10.10.10.12`). Após a correção, a rede voltou.

**Kerberos e horário.** O fuso horário do Windows Server foi corrigido antes da criação do domínio, já que o Kerberos recusa autenticação com diferença de relógio acima de 5 minutos.

---

## 🧭 Próximos passos

- Desativar login SSH por senha (somente chave)
- Avaliar o AdGuard para toda a rede, com plano de contingência
- Backup automatizado em HD externo
- Wazuh coletando logs do Linux e do Windows
- Laboratório ofensivo em rede isolada (Kali × alvos vulneráveis)

---

## 🛠️ Tecnologias

Proxmox VE · Debian 13 · LXC · KVM/QEMU · AdGuard Home · Samba · Jellyfin · Windows Server 2025 · Active Directory · DNS · PowerShell · Bash · SSH · TOTP
