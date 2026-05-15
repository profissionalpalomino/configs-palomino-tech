# Configuração de Automação — Palomino Tech

Explica tudo que foi configurado para o Claude conseguir fazer deploy no servidor, push no GitHub e edições de código **de forma totalmente automática**, sem precisar digitar senha ou confirmar nada manualmente.

---

## Visão geral

Quando o Claude executa um comando como:
```
scp arquivo.html root@187.77.57.158:/caminho/
```
ou:
```
git push origin master
```

…ele não pede senha porque dois mecanismos estão configurados: **chave SSH** (para o servidor) e **Windows Credential Manager** (para o GitHub). Veja abaixo como cada um funciona.

---

## 1. Acesso ao servidor via SSH sem senha

### O que é
Uma chave SSH é um par de arquivos: **chave privada** (fica no seu computador) e **chave pública** (fica no servidor). Quando o Claude tenta conectar, o servidor verifica automaticamente se a chave bate — sem precisar de senha.

### Onde estão os arquivos
| Arquivo | Caminho | Função |
|---|---|---|
| Chave privada | `C:\Users\rodri\.ssh\id_rsa` | Fica só no seu PC. Nunca compartilhe. |
| Chave pública | `C:\Users\rodri\.ssh\id_rsa.pub` | Cópia enviada para o servidor |

A chave foi criada em **29/04/2026**.

### Como a chave pública foi instalada no servidor
A chave pública (`id_rsa.pub`) foi adicionada ao arquivo de chaves autorizadas do servidor:
```
/root/.ssh/authorized_keys
```

A partir daí, qualquer conexão SSH ou SCP vinda deste PC é aceita automaticamente pelo servidor `root@187.77.57.158`.

### Como recriar do zero (se precisar)
```powershell
# 1. Gerar novo par de chaves no PC
ssh-keygen -t rsa -b 4096 -f C:\Users\rodri\.ssh\id_rsa

# 2. Copiar a chave pública para o servidor (vai pedir senha uma vez)
ssh-copy-id root@187.77.57.158

# 3. Testar (deve entrar sem pedir senha)
ssh root@187.77.57.158 "echo OK"
```

---

## 2. Push no GitHub sem senha

### O que é
O **Windows Credential Manager** é um cofre de senhas nativo do Windows. O Git está configurado para usar esse cofre como `credential.helper`.

### Como está configurado
```
credential.helper=manager
user.name=Rodrigo Palomino
user.email=profissionalpalomino@gmail.com
```

Da primeira vez que foi feito `git push`, o Windows abriu uma janela pedindo login no GitHub. Após autenticar, o token de acesso foi salvo no Credential Manager. Todos os pushes seguintes usam esse token automaticamente — sem janela, sem senha.

### Como verificar
```powershell
# Ver configurações do git
git config --list | Select-String "credential|user"

# Ver credenciais salvas no Windows
cmdkey /list | Select-String "git"
```

### Como recriar do zero (se precisar)
```powershell
# 1. Remover a credencial antiga
cmdkey /delete:LegacyGeneric:target=git:https://github.com

# 2. Fazer qualquer push — o Windows vai pedir login novamente
git push origin master
# (vai abrir o navegador para autenticar com o GitHub)
```

---

## 3. Estrutura do servidor

| Item | Valor |
|---|---|
| IP | `187.77.57.158` |
| Usuário | `root` |
| Gerenciador | Easypanel |
| Orquestrador | Docker Swarm |
| Nome do projeto | `palomino-site` |
| Nome do serviço | `palomino-site_barbearias-finder` |

### Caminho dos arquivos no servidor
```
/etc/easypanel/projects/palomino-site/barbearias-finder/code/
  public/
    proposta/       ← proposta Barbearias Finder
    logos/          ← página de seleção de logos
  Dockerfile
  server.js
  ...
```

### URL pública base
```
https://barbearias.profissionalpalomino.cloud/
```

---

## 4. Fluxo completo de deploy

Toda vez que um arquivo é alterado, o Claude executa essa sequência:

```powershell
# Passo 1 — enviar o arquivo para o servidor
scp arquivo.html root@187.77.57.158:/etc/easypanel/projects/palomino-site/barbearias-finder/code/public/pasta/

# Passo 2 — reconstruir a imagem Docker e atualizar o serviço
ssh root@187.77.57.158 "cd /etc/easypanel/projects/palomino-site/barbearias-finder/code && docker build -t barbearias-finder:latest . -q && docker service update --force --image barbearias-finder:latest palomino-site_barbearias-finder"
```

O Docker Swarm faz a substituição do container sem derrubar o site (zero downtime).

---

## 5. Repositórios no GitHub

| Repo | URL | Branch | Finalidade |
|---|---|---|---|
| barbearias-finder | `github.com/profissionalpalomino/barbearias-finder` | `master` | Código do produto |
| palomino-proposta-template | `github.com/profissionalpalomino/palomino-proposta-template` | `master` | Template de proposta reutilizável |
| configs-palomino-tech | `github.com/profissionalpalomino/configs-palomino-tech` | `master` | Steering files, rules e automação |

### Fluxo de commit e push
```powershell
git add arquivo.html
git commit -m "descrição da mudança"
git push origin master
```

---

## 6. Configuração do Claude (para automação total)

O Claude acessa tudo isso via **Claude Code** (CLI), que roda comandos PowerShell e SSH diretamente na sua máquina. Não há nada adicional instalado — tudo usa as ferramentas nativas do Windows + as credenciais já configuradas acima.

Para o Claude conseguir fazer deploy automático, basta que:
- [ ] O PC tenha a chave SSH em `C:\Users\rodri\.ssh\id_rsa`
- [ ] O GitHub esteja autenticado no Windows Credential Manager
- [ ] O servidor `187.77.57.158` esteja acessível via internet
- [ ] O Docker Swarm esteja rodando no servidor (gerenciado pelo Easypanel)

---

## 7. Segurança — o que saber

- A **chave privada SSH** (`id_rsa`) nunca deve ser enviada para o GitHub ou compartilhada
- O **token do GitHub** fica no Credential Manager do Windows — protegido pela conta do Windows
- O acesso ao servidor é feito como `root` — o Claude só executa comandos relacionados ao projeto
- Se quiser revogar o acesso do Claude: remova a chave pública de `/root/.ssh/authorized_keys` no servidor e delete a credencial do GitHub no Credential Manager

---

*Arquivo gerado automaticamente pela sessão de configuração — Palomino Tech*
