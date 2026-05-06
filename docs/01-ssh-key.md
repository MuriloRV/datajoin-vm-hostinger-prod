# 01 — SSH key

Gera o par de chaves local pra acessar a VM Hostinger sem senha, adiciona a pública na VM, e configura o alias `vm-hostinger` no `~/.ssh/config` da workstation.

## 1. Gerar o par de chaves (workstation Windows / PowerShell)

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_vm_hostinger" -C "vm-hostinger-prod"
```

- `-t ed25519` — algoritmo moderno, chave curta e rápida (preferir sobre RSA).
- `-f` — caminho/nome do arquivo. Gera `id_ed25519_vm_hostinger` (privada) e `id_ed25519_vm_hostinger.pub` (pública) em `C:\Users\<user>\.ssh\`. Convenção `id_ed25519_<contexto>` já usada nas outras chaves (`id_ed25519_github`, `id_ed25519_bitbucket`, etc).
- `-C` — comment field (vai no final da chave pública, útil pra identificar nos `authorized_keys` da VM).

**Passphrase:** ssh-keygen vai perguntar. Recomendado **definir uma passphrase** (qualquer atacante que rouba o arquivo da chave precisa quebrar a passphrase também). No Windows, o `ssh-agent` (serviço `OpenSSH Authentication Agent`) cacheia a passphrase pra você não digitar a cada conexão:

```powershell
# Habilitar e iniciar o agent (admin PowerShell, 1x)
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# Adicionar a chave ao agent (digita passphrase 1x)
ssh-add "$env:USERPROFILE\.ssh\id_ed25519_vm_hostinger"
```

## 2. Adicionar a chave pública na VM

Na primeira conexão, ainda dá pra usar senha do `root` (a que a Hostinger forneceu no painel após a reinstalação). A partir daí, copiar a pública pro `authorized_keys`.

**Opção A — `ssh-copy-id` (se disponível no Git Bash / WSL):**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_vm_hostinger.pub root@<VM_IP>
```

**Opção B — manual via PowerShell (sem ssh-copy-id):**
```powershell
$pub = Get-Content "$env:USERPROFILE\.ssh\id_ed25519_vm_hostinger.pub"
ssh root@<VM_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '$pub' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

(digita a senha do root da Hostinger uma única vez)

## 3. Configurar alias no `~/.ssh/config` da workstation

Append do snippet em [`../ssh/config-snippet`](../ssh/config-snippet) ao seu `C:\Users\<user>\.ssh\config` (criar arquivo se não existir).

```powershell
Get-Content .\ssh\config-snippet | Add-Content "$env:USERPROFILE\.ssh\config"
```

Depois disso:
```powershell
ssh vm-hostinger
```
deve logar direto, sem senha, sem digitar IP.

## 4. Verificação

```powershell
ssh vm-hostinger "hostname && whoami && uname -a"
```

Saída esperada: hostname da VM Hostinger + `root` + kernel Linux.

## Próximo passo

Depois que SSH com chave estiver funcionando, o próximo passo será desabilitar login com senha do SSH (hardening) — vai virar `02-ssh-hardening.md`.
