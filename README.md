# CT-Ansible

Ansible playbooks and roles for provisioning CT infrastructure — multi-node Proxmox cluster, Docker hosts, Cloudflare tunnels, and shared storage across two locations.

## Infrastructure Overview

```
                   LAN 192.168.0.0/24
                          |
          +---------------+----------------+
          |                                |
    macpro-pve (.120)                pve-2 (.100)
    Proxmox VE node                  Proxmox VE node
    ├── shared_storage (Samba+virtiofs)  ├── cloudflare_tunnel
    └── DevOps VMs:                      └── docker-host LXC (.139)
        ├── ubuntu-devops (.101)
        ├── freebsd-devops (.102)
        └── windows-devops (.103)
```

## Wymagania

- Ansible >= 2.15
- Python 3.x na control node
- Dostep SSH do wszystkich hostow (klucz w `.ansible_priv_key`)
- Ansible Vault password w `.vault_pass`

## Szybki start

```bash
# Zainstaluj kolekcje Galaxy
ansible-galaxy install -r requirements.yml

# Sprawdz polaczenie
ansible all -m ping

# Uruchom caly playbook
ansible-playbook playbook.yml

# Tylko wybrany host
ansible-playbook playbook.yml --limit macpro-pve

# Tylko wybrana rola (tag = nazwa roli)
ansible-playbook playbook.yml --limit pve-2 --tags cloudflare_tunnel
```

## Struktura projektu

```
.
├── ansible.cfg              # Konfiguracja Ansible (inventory, SSH, Vault)
├── inventory.yml            # Hosty i grupy
├── playbook.yml             # Glowny playbook - orkiestracja rol
├── requirements.yml         # Kolekcje Galaxy (community.general, community.proxmox, ansible.posix)
├── ssh_config               # SSH config dla hostow
├── group_vars/
│   ├── all/main.yml         # Wspolne zmienne (users, DNS)
│   ├── docker/main.yml      # Zmienne grupy docker
│   ├── proxmox/main.yml     # Zmienne grupy proxmox
│   └── ubuntu/              # Zmienne grupy ubuntu
├── host_vars/
│   ├── docker-host/         # Zmienne per-host
│   ├── macpro-pve/
│   ├── pve-2/
│   └── ubuntu-lab/
└── roles/
    ├── proxmox/             # Repo cleanup (enterprise->no-sub), narzedzia, aktualizacje
    ├── hostname/            # Ustawienie hostname + /etc/hosts
    ├── login/               # Uzytkownicy, klucze SSH
    ├── security/            # fail2ban, UFW, unattended-upgrades
    ├── shared_storage/      # Samba (Windows) + virtiofs mount points
    ├── cloudflare_tunnel/   # cloudflared install + service
    └── docker/              # Docker engine install + user groups
```

## Role

| Rola | Hosty | Opis |
|------|-------|------|
| `proxmox` | proxmox | Usuwa enterprise repo, dodaje no-subscription, instaluje narzedzia (htop, ncdu, jq...) |
| `hostname` | all | Ustawia hostname z inventory + aktualizuje `/etc/hosts` |
| `login` | all | Tworzy uzytkownikow, konfiguruje klucze SSH |
| `security` | all | fail2ban (maxretry=5, bantime=1h), UFW (deny incoming, allow SSH), unattended-upgrades |
| `shared_storage` | macpro-pve | Samba share `devops-shared` dla Windows, katalog virtiofs dla Linux/FreeBSD |
| `cloudflare_tunnel` | pve-2 | Instaluje cloudflared z oficjalnego repo, rejestruje tunel jako systemd service |
| `docker` | docker, ubuntu-devops | Instaluje Docker engine, dodaje usera do grupy docker |

## Kolejnosc wykonania (playbook.yml)

1. **proxmox** -- naprawa repozytoriow zanim `security` uruchomi `apt`
2. **hostname + login + security** -- bazowa konfiguracja wszystkich hostow
3. **shared_storage** -- Samba + virtiofs na macpro-pve
4. **cloudflare_tunnel** -- cloudflared na pve-2
5. **docker** -- Docker na LXC i ubuntu-devops
6. **devops** -- mount virtiofs share na VM-kach DevOps (inline tasks)

## Vault

Sekrety (tokeny Cloudflare, hasla) sa przechowywane w zaszyfrowanych plikach Ansible Vault. Plik z haslem do Vault wskazany w `ansible.cfg`:

```ini
vault_password_file = .vault_pass
```

Aby edytowac sekrety:

```bash
ansible-vault edit host_vars/pve-2/secrets.yml
```

## Siec i storage

- **virtiofs** -- bezposredni mount z hypervisora do VM (Linux/FreeBSD), zero overhead sieciowego
- **Samba (SMB)** -- dla Windows VM (`\\192.168.0.120\devops-shared`)
- **UFW** -- port 445/tcp otwarty tylko dla LAN (192.168.0.0/24) i VLAN (10.10.10.0/24)

## Powiazane repozytoria

| Repo | Opis |
|------|------|
| [CT-Terraform](../CT-Terraform) | Infrastruktura jako kod: Proxmox bootstrap, Cloudflare DNS (11 stref), tunele CF, Uptime Kuma, Nginx Proxy Manager |

Terraform tworzy infrastrukture (VM, sieci, DNS, tunele), Ansible ja konfiguruje (OS, pakiety, uslugi).

## Pliki wrazliwe (nie commituj!)

- `.ansible_priv_key` -- klucz SSH
- `.vault_pass` -- haslo Ansible Vault
- `host_vars/*/secrets.yml` -- zaszyfrowane sekrety per-host
