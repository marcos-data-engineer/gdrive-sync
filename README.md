# gdrive-sync

Sincronização bidirecional entre um diretório Linux local e o Google Drive — gerenciada por um único script interativo em Bash.

A ferramenta utiliza [rclone bisync](https://rclone.org/bisync/), [inotify-tools](https://github.com/inotify-tools/inotify-tools), systemd e cron para manter a sincronização contínua entre um diretório local e uma pasta no Google Drive. Alterações locais são enviadas em poucos segundos; alterações remotas são baixadas conforme um agendamento configurável.

Arquivo local salvo  ──►  inotify detecta  ──►  debounce (5s)  ──►  rclone bisync  ──►  Drive atualizado

Cron executa (N min) ──►  rclone bisync    ──►  Diretório local atualizado


---

## Funcionalidades

- **Arquivo único e sem dependências manuais** — um arquivo `.sh` que instala o `rclone` e o `inotify-tools` automaticamente.
- **Menu interativo** — adicione, liste, edite, remova e monitore rotinas de sincronização sem memorizar comandos.
- **Sincronização bidirecional** — sincronia completa local ↔ Drive via `rclone bisync`.
- **Envio instantâneo (Local → Drive)** — o `inotifywait` monitora alterações em tempo real com intervalo de estabilização (*debounce* de 5s).
- **Download agendado (Drive → Local)** — o cron busca atualizações remotas no intervalo configurado.
- **Resolução de conflitos** — o arquivo mais recente prevalece (`--conflict-resolve newer`); versões antigas são renomeadas e preservadas.
- **Controle de bloqueio e trava (Lock & Debounce)** — previne execuções sobrepostas e bloqueios por limite de requisições na API do Google.
- **Rotação automática de logs** — logs são rotacionados ao atingir 50 MB.
- **Gerenciamento via Systemd** — o serviço de monitoramento persiste após reinicializações do sistema; controle via `systemctl`.
- **Múltiplos perfis** — gerencie vários pares independentes de pastas/Drive simultaneamente.
- **Segurança e isolamento** — execução em nível de usuário (nunca como root), limpeza de cache individualizada e endurecimento de permissões no systemd.

---

## Requisitos do Sistema

| Requisito | Notas |
|---|---|
| **Sistema Operacional** | Fedora 38+, Ubuntu 20.04+, Debian 11+ ou qualquer distribuição Linux com systemd |
| **Bash** | Versão 4.4+ |
| **Acesso à Internet** | Necessário para comunicação com a API do Google Drive |
| **Conta Google** | Com acesso à pasta destino no Drive |
| **Projeto no Google Cloud** | Gratuito — consulte o [guia de configuração abaixo](#configuração-de-credenciais-na-api-do-google) |

Todas as dependências secundárias (`rclone`, `inotify-tools`, `curl`) são verificadas e instaladas automaticamente na primeira execução.

---

## Início Rápido

```bash
# Clonar o repositório
git clone [https://github.com/marcos-data-engineer/gdrive-sync.git](https://github.com/marcos-data-engineer/gdrive-sync.git)
cd gdrive-sync

# Conceder permissão de execução
chmod +x gdrive-sync.sh

# Executar o gerenciador
./gdrive-sync.sh
Antes de adicionar o primeiro perfil de sincronização, obtenha as credenciais da API do Google conforme o guia a seguir.

Configuração de Credenciais na API do Google
É necessário obter um Client ID e um Client Secret criando um aplicativo do tipo "Desktop App" no Google Cloud Console. Essas credenciais autorizam o rclone a se conectar à sua conta do Google Drive.

Passo 1 — Criar um Projeto no Google Cloud
Acesse o Google Cloud Console.

Faça login com a conta Google que possui acesso à pasta do Drive.

Clique no menu suspenso de projetos no canto superior esquerdo.

Clique em Novo Projeto.

Defina um nome (ex: gdrive-sync) e clique em Criar.

Garanta que o novo projeto esteja selecionado no menu superior.

Passo 2 — Ativar a Google Drive API
No menu lateral esquerdo, navegue até APIs e Serviços → Biblioteca.

Link direto: https://console.cloud.google.com/apis/library

Pesquise por Google Drive API.

Selecione a API e clique no botão Ativar.

Passo 3 — Configurar a Tela de Consentimento OAuth
Acesse APIs e Serviços → Tela de consentimento OAuth.

Link direto: https://console.cloud.google.com/apis/credentials/consent

Selecione o tipo de usuário Externo e clique em Criar.

Preencha os campos obrigatórios:

Nome do app: gdrive-sync

E-mail para suporte do usuário: Seu e-mail

Dados de contato do desenvolvedor: Seu e-mail

Clique em Salvar e continuar.

Na etapa de Escopos, clique em Adicionar ou remover escopos:

Filtre por drive.

Marque a opção .../auth/drive (Ver, editar, criar e excluir todos os seus arquivos do Google Drive).

Clique em Atualizar e depois em Salvar e continuar.

Na etapa de Usuários de teste, clique em + Add Users:

Adicione o seu endereço de e-mail do Gmail / Google Workspace.

Clique em Adicionar e depois em Salvar e continuar.

Passo 4 — Criar o Client ID e Client Secret
Acesse APIs e Serviços → Credenciais.

Link direto: https://console.cloud.google.com/apis/credentials

Clique em + Criar credenciais no topo e escolha ID do cliente OAuth.

No campo Tipo de aplicativo, escolha App para Computador.

Informe um nome (ex: gdrive-sync-client) e clique em Criar.

O painel exibirá o seu Client ID e o seu Client Secret. Copie ambos os valores.

Passo 5 — Obter o ID da Pasta do Google Drive
Abra o Google Drive no navegador.

Acesse a pasta que deseja sincronizar.

Copie o código exibido no final da URL após /folders/:

Plaintext
[https://drive.google.com/drive/folders/1aBcDeFgHiJkLmNoPqRsTuVwXyZ012345](https://drive.google.com/drive/folders/1aBcDeFgHiJkLmNoPqRsTuVwXyZ012345)
                                        └───────────── Folder ID ─────────────┘
Passo 6 — Executar a Sincronização
Execute o script e selecione a opção 1) Add new sync:

Bash
./gdrive-sync.sh
Cole o Client ID, Client Secret e Folder ID quando solicitado. O script abrirá o navegador para confirmar a autorização do dispositivo.

Modo de Uso
Menu Interativo
Bash
./gdrive-sync.sh
Plaintext
 ┌─────────────────────────────────────────────┐
 │    Google Drive Sync Manager  v0.2.0         │
 │    rclone bisync + inotify-tools            │
 └─────────────────────────────────────────────┘

 Perfis ativos:
   ● meus-documentos
   ● backups-projetos

 Menu
     1) Adicionar nova sincronização   2) Listar configurações
     3) Exibir status dos serviços     4) Editar configuração
     5) Remover uma configuração       6) Visualizar logs
     7) Executar sincronização manual  8) Testar conexão com o Drive
     9) Configurações globais         10) Executar testes de integridade
     0) Sair
Comandos Via CLI (Não-Interativo)
Bash
# Verificação rápida de status
./gdrive-sync.sh --status

# Teste de conexão com o Drive para um perfil específico
./gdrive-sync.sh --test-drive meus-documentos

# Simulação (Dry-run)
./gdrive-sync.sh --dry-run

# Modo verboso com diagnósticos estendidos
./gdrive-sync.sh -v

# Ajuda e versão
./gdrive-sync.sh --help
./gdrive-sync.sh --version
Gerenciamento de Serviços via Systemd
Cada perfil criado gera um serviço systemd dedicado no usuário:

Bash
# Verificar status do serviço
systemctl --user status gdrive-sync-meus-documentos

# Parar a sincronização
systemctl --user stop gdrive-sync-meus-documentos

# Reiniciar a sincronização
systemctl --user restart gdrive-sync-meus-documentos

# Acompanhar logs em tempo real
journalctl --user -u gdrive-sync-meus-documentos -f
Arquitetura do Sistema
Plaintext
┌──────────────────────────────────────────────────────────────┐
│  gdrive-sync.sh (Gerenciador Interativo)                     │
│                                                              │
│  Configura:                                                  │
│   ├── Remote no rclone (Autenticação OAuth + ID da pasta)   │
│   ├── Unidade do Systemd (Serviço do monitor inotify)        │
│   └── Tarefa no Cron (Pull periódico do Drive)              │
└──────────────────────────────────────────────────────────────┘

┌─ Serviço Systemd ────────────────────────────────────────────┐
│                                                              │
│  inotifywait -m -r (Monitora alterações no diretório local) │
│       │                                                      │
│       ├── Alteração detectada                                │
│       ├── Debounce (Aguarda 5s para estabilização)           │
│       ├── Valida trava (Lock)                                │
│       └── Executa rclone bisync local ↔ remoto               │
└──────────────────────────────────────────────────────────────┘

┌─ Tarefa Cron ────────────────────────────────────────────────┐
│                                                              │
│  A cada N minutos:                                           │
│    Executa rclone bisync local ↔ remoto                      │
│    (Sincroniza alterações vindas da web/mobile no Drive)     │
└──────────────────────────────────────────────────────────────┘
Resolução de Conflitos
Quando um mesmo arquivo é modificado em ambas as pontas entre os ciclos de sincronização:

O arquivo com a data de alteração mais recente é mantido (--conflict-resolve newer).

A versão anterior é salva com um sufixo numérico de segurança (--conflict-loser num), por exemplo: relatorio.docx.conflict1.

Nenhuma alteração é sobrescrita sem backup automático.

Licença
Este projeto está licenciado sob a Licença MIT — consulte o arquivo LICENSE para obter mais detalhes.


<FollowUp>
O README traduzido e o arquivo da imagem PNG gerado no Fedora atenderam totalmente às necessidades do projeto?
</FollowUp>
