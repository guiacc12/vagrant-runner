# Guia de Configuração CI/CD - GitHub Actions + Self-Hosted Runner

## Pré-requisitos
- VM local rodando Linux (Ubuntu/Debian recomendado)
- Nginx instalado na VM
- Git instalado na VM
- Conta no GitHub

---

## 1. Criar Repositório no GitHub

### No seu PC:
```bash
# Crie uma pasta para o projeto
mkdir meu-site-cicd
cd meu-site-cicd

# Crie a estrutura de pastas
mkdir -p .github/workflows

# Adicione os arquivos index.html e .github/workflows/deploy.yml
# (use os arquivos que criei nos artifacts)

# Inicialize o repositório
git init
git add .
git commit -m "Configuração inicial do projeto"

# Crie o repositório no GitHub e adicione o remote
git remote add origin https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
git branch -M main
git push -u origin main
```

---

## 2. Configuração da VM Local

### Instalar dependências necessárias
```bash
# Atualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Nginx e Git
sudo apt install nginx git -y

# Iniciar e habilitar Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Clonar o repositório na VM
```bash
# Remover conteúdo padrão do nginx
sudo rm -rf /var/www/html/*

# Clonar seu repositório
cd /var/www
sudo git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git html

# Ajustar permissões
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

# Configurar git para permitir operações neste diretório
cd /var/www/html
sudo git config --global --add safe.directory /var/www/html
```

---

## 3. Instalar GitHub Actions Runner na VM

### Acessar as configurações do Runner no GitHub

1. Vá no seu repositório no GitHub
2. Clique em **Settings** → **Actions** → **Runners**
3. Clique em **New self-hosted runner**
4. Escolha **Linux** e **x64**
5. **Copie os comandos que aparecem na tela**

### Executar os comandos na VM

```bash
# Os comandos serão parecidos com estes (use os que o GitHub mostrou!):

# Criar diretório para o runner
mkdir actions-runner && cd actions-runner

# Baixar o runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extrair
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# IMPORTANTE: Se estiver rodando como root, execute este comando:
export RUNNER_ALLOW_RUNASROOT="1"

# Configurar o runner
./config.sh --url https://github.com/SEU_USUARIO/SEU_REPOSITORIO --token SEU_TOKEN_DO_GITHUB

# Durante a configuração:
# - Nome do runner: pode deixar o padrão ou colocar "vm-local"
# - Labels: aperte Enter (usa o padrão)
# - Work folder: aperte Enter (usa o padrão)
```

### Instalar como serviço (para rodar sempre)

```bash
# Instalar como serviço do sistema
sudo ./svc.sh install

# Iniciar o serviço
sudo ./svc.sh start

# Verificar status
sudo ./svc.sh status
```

**Pronto!** O runner está rodando e vai executar automaticamente quando você fizer push!

---

## 4. Configurar Permissões Sudo na VM

Para o runner executar comandos sudo sem senha:

```bash
# Na VM, edite o sudoers
sudo visudo

# Adicione no final do arquivo (substitua SEU_USUARIO pelo usuário que instalou o runner):
SEU_USUARIO ALL=(ALL) NOPASSWD: /usr/bin/git, /usr/bin/systemctl
```

**OU** configure para o usuário do runner:

```bash
# Descubra qual usuário está rodando o runner
ps aux | grep Runner.Listener

# Adicione esse usuário no sudoers com as permissões necessárias
```

---

## 5. Testar o CI/CD

```bash
# No seu PC, faça uma alteração no index.html
# Por exemplo, mude a versão para 1.0.1

git add .
git commit -m "Teste de deploy automático"
git push origin main
```

Vá em **GitHub → Actions** e acompanhe o deploy!

---

## Como Funciona

1. **Você faz push** → Código vai pro GitHub
2. **GitHub Actions detecta** → Procura um runner disponível
3. **Runner na VM executa** → Roda os comandos localmente
4. **Site atualizado** → Nginx serve o novo conteúdo!

---

## Vantagens desta Abordagem

- Sem SSH necessário - Runner executa localmente
- Sem secrets complexos - Não precisa chaves SSH
- Sem port forwarding - Não precisa expor sua VM
- Sem ngrok/túneis - Runner se conecta ao GitHub
- Mais rápido - Não tem latência de rede
- Mais simples - Menos coisas para configurar
- Mais seguro - VM não fica exposta

---

## Solução de Problemas

### Erro: "Your local changes to the following files would be overwritten by merge"

Este é um erro **muito comum** que acontece quando você editou arquivos diretamente na VM.

**IMPORTANTE:** Na VM você **NÃO deve editar arquivos manualmente**! Edite sempre no seu PC e faça push.

**Solução (escolha uma):**

```bash
# Opção 1: Descartar mudanças locais (RECOMENDADO)
cd /var/www/html
sudo git reset --hard
sudo git pull origin main

# Opção 2: Salvar mudanças locais
cd /var/www/html
sudo git add .
sudo git commit -m "Mudanças locais"
sudo git pull origin main

# Opção 3: Guardar temporariamente (stash)
cd /var/www/html
sudo git stash
sudo git pull origin main
sudo git stash pop  # Para recuperar depois (opcional)
```

### Runner não aparece como "online" no GitHub
```bash
# Verifique o status
cd ~/actions-runner
sudo ./svc.sh status

# Veja os logs
journalctl -u actions.runner.* -f

# Reinicie o serviço
sudo ./svc.sh stop
sudo ./svc.sh start
```

### Deploy falha com "Permission denied"
```bash
# Verifique as permissões do diretório
ls -la /var/www/html

# Ajuste se necessário
sudo chown -R $USER:$USER /var/www/html

# Ou para o usuário www-data (nginx)
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

### Git pull falha com "dubious ownership"
```bash
# Configure git para confiar no diretório
cd /var/www/html
sudo git config --global --add safe.directory /var/www/html

# Configure credenciais se necessário (para repos privados)
git config --global credential.helper store
```

### Runner para de funcionar após reiniciar VM
```bash
# Certifique-se que está instalado como serviço
cd ~/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start

# Habilite para iniciar automaticamente
sudo systemctl enable actions.runner.*

# Verifique se está ativo
sudo systemctl status actions.runner.*
```

### Nginx não mostra as atualizações
```bash
# Limpe o cache do navegador (Ctrl + F5)

# Ou force reload do Nginx
sudo systemctl reload nginx

# Verifique se os arquivos foram atualizados
ls -lah /var/www/html/
cat /var/www/html/index.html

# Veja os logs do Nginx
sudo tail -f /var/log/nginx/error.log
```

---

## Comandos Úteis do Runner

```bash
# Ver status
sudo ./svc.sh status

# Parar runner
sudo ./svc.sh stop

# Iniciar runner
sudo ./svc.sh start

# Reiniciar runner
sudo ./svc.sh stop
sudo ./svc.sh start

# Desinstalar runner
sudo ./svc.sh uninstall

# Ver logs em tempo real
journalctl -u actions.runner.* -f
```

---

## Conclusão

Agora toda vez que você fizer push para o GitHub, o deploy acontecerá **automaticamente e localmente** na sua VM!

**Acesse:** `http://IP_DA_SUA_VM`

Muito mais simples que SSH, túneis e port forwarding!