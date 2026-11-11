# Windows Server (por alto)

Anotado só para reconhecer os nomes e consultar no futuro. Fora do foco: os projetos usam Linux, Docker e Kubernetes, sem IIS. O que não vem das páginas da Microsoft consultadas é de memória e está marcado.

## O que é

Windows Server é o Windows feito para servidor. A diferença que importa é como ele é instalado (doc):

| Opção | O que é |
|---|---|
| **Server Core** | instalação mínima, sem área de trabalho (sem GUI, por design). Menor, com menos código, e por isso com menor superfície de ataque |
| **Desktop Experience** | com interface gráfica completa; instala muitos serviços que normalmente não se usa |

O Server Core se administra de longe: linha de comando, PowerShell, ou ferramentas gráficas rodando em outra máquina (RSAT e Windows Admin Center, citados na doc). É a mesma lógica da imagem enxuta do Docker: menos coisa instalada, menos risco.

## Linux e Windows lado a lado

Os conceitos são os mesmos; mudam os nomes e as ferramentas. A tabela é de memória, exceto onde a doc é citada.

| Assunto | Linux | Windows Server |
|---|---|---|
| Administrar de longe | SSH | PowerShell remoto, RDP (área de trabalho remota) |
| Usuários | lista do sistema (`/etc`) | usuários locais e, na empresa, o Active Directory (cadastro central de usuários) |
| Permissões | dono, grupo e `rwx` | ACL: lista de quem pode o quê em cada pasta, mais detalhada |
| Manter um programa de pé | systemd | Serviços do Windows |
| Firewall | iptables, ufw | Windows Defender Firewall |
| Logs | journal, `/var/log` | Event Viewer (a doc cita o `Wevtutil` para consultar eventos) |
| Processos | `ps` | Gerenciador de Tarefas (`Taskmgr`, citado na doc) |
| Servidor web | Nginx | IIS |

## IIS, o servidor web do Windows

Conceitos da doc de arquitetura do IIS:

- **HTTP.sys:** o ouvinte de HTTP e HTTPS. Fica no kernel do Windows e recebe os pedidos antes de tudo.
- **Application pool:** um grupo de processos isolados onde rodam as aplicações. Uma aplicação travada não derruba as outras, porque cada pool tem os seus processos (worker process).
- **Módulos:** o IIS é montado por peças (autenticação, compressão, cache, logs...). Instala-se só o que precisa, para reduzir a superfície de ataque, a mesma lógica do Server Core.
- **Site e binding** (de memória): o site é a aplicação publicada, e o binding diz em qual IP, porta e nome do site ele responde. É o que faz várias aplicações na mesma porta, cada uma com seu DNS.

O caminho de um pedido (resumo da doc):

```text
navegador ──> HTTP.sys (kernel) ──> o IIS lê a configuração
                                       │
                          escolhe o application pool ──> worker process
                                                              │
                                    a aplicação processa ──> resposta volta
```

É o mesmo desenho do `Nginx → API` de [linux-processos-servicos-e-rede.md](linux-processos-servicos-e-rede.md):

| Nginx | IIS |
|---|---|
| `server_name api.galo.exemplo.com` | binding com o nome do site |
| `proxy_pass http://galo-api:8080` | o site/aplicação dentro do pool |
| bloco `server {}` | site |

No .NET moderno, o IIS costuma ficar na frente do Kestrel, o servidor da própria aplicação (de memória).

## Windows containers

Os containers dos projetos são Linux. Existem também containers de Windows, e a regra é a mesma de sempre (doc): o container compartilha o kernel do host e tem um pedaço dos arquivos do sistema (a imagem base). Um container Windows precisa de host Windows e de imagem Windows; não roda em Linux.

Imagens base da Microsoft (doc), da maior para a menor:

| Imagem | O que é |
|---|---|
| **Windows** | o conjunto completo de APIs e serviços, menos os papéis de servidor |
| **Windows Server** | o conjunto completo de APIs e serviços |
| **Server Core** | subconjunto menor, com o .NET Framework completo |
| **Nano Server** | a menor, com suporte ao .NET moderno (.NET Core) |

A imagem do .NET moderno é construída sobre o Nano Server (doc), mas as que os projetos usam (`mcr.microsoft.com/dotnet/aspnet`) têm variantes Linux, que são as do caminho.

Quando se usa container Windows (de memória): aplicações .NET Framework antigas, que só rodam em Windows. Para o .NET moderno, o container Linux é a escolha. O Docker Desktop do Windows roda os containers Linux dentro do WSL2. O Kubernetes também suporta nodes Windows (seção 13 do mapa da doc, deixada de fora).

## O que isso significa

- Os projetos: Linux, Docker e Kubernetes, sem IIS.
- O Windows Server aparece quando o cliente ou a empresa já o tem (aplicações legadas, Active Directory). Basta reconhecer os nomes: Server Core, IIS, application pool, Active Directory, ACL.
- O raciocínio de permissão, rede, firewall e logs é o mesmo; muda a ferramenta.

## Deixado de fora por decisão

Instalação e configuração do IIS, PowerShell em detalhe, Active Directory e Hyper-V. Decisão do usuário: não será usado agora nem no futuro; os links abaixo são para o dia em que for preciso.

## Links para pesquisar no dia que precisar

Consultados nesta sessão:

- [O que é o Server Core](https://learn.microsoft.com/en-us/windows-server/administration/server-core/what-is-server-core)
- [Arquitetura do IIS](https://learn.microsoft.com/en-us/iis/get-started/introduction-to-iis/introduction-to-iis-architecture)
- [Sobre os Windows containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/)

Não consultados (endereços de memória; confira se abrem):

- [Documentação do IIS](https://learn.microsoft.com/en-us/iis/)
- [Hospedar o ASP.NET Core no IIS](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/)
- [Documentação do Windows Server](https://learn.microsoft.com/en-us/windows-server/)
- [Windows Admin Center](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/overview)
- [PowerShell](https://learn.microsoft.com/en-us/powershell/)
- [Windows containers (documentação)](https://learn.microsoft.com/en-us/virtualization/windowscontainers/)
- [Windows no Kubernetes](https://kubernetes.io/docs/concepts/windows/)
