# SEMED — Gestão de documentos por setor

Primeira versão funcional para testes, com interface responsiva e backend. Um projeto separado do portal público de apresentação.

## O que funciona

- Configuração inicial com a criação da conta administrativa. Não há senha padrão em produção.
- Administrador cadastra setores e funcionários. A conta administrativa não consulta documentos dos setores.
- Funcionários têm conta individual, senha pessoal e vínculo a exatamente um setor.
- Cada conta de funcionário pode listar, baixar, enviar e editar documentos do próprio setor. Não existe acesso cruzado entre áreas, inclusive por URL direta ou histórico de versões.
- Senha temporária exige troca no primeiro acesso.
- Limite de tentativas de login. Desativação de conta e redefinição de senha encerram as sessões.
- Arquivos privados em R2 e metadados, contas, sessões e permissões em D1.
- Documentos organizados por categoria, busca local no resultado já autorizado pelo servidor, novas versões, lixeira e restauração.
- Registro de operações do setor. A área administrativa vê apenas as operações administrativas.

## Tecnologia e estrutura

React/TypeScript e CSS para a interface. Vinext e Cloudflare Workers para a API. Drizzle descreve as tabelas e gera migrações SQLite/D1. Nenhuma senha ou documento fica em localStorage. Apenas um cookie HttpOnly referencia a sessão, cujo token é armazenado como SHA-256 no banco.

- `app/page.tsx`: interface de acesso e área de trabalho.
- `app/globals.css`: apresentação responsiva.
- `app/api/[...path]/route.ts`: entrada da API.
- `lib/api.ts`: autenticação, autorização, operações de contas e documentos.
- `lib/security.ts`: hash de senha, tokens e comparação.
- `db/schema.ts`, `drizzle/`: esquema e migrações.
- `tests/access-control.mjs`: testes de integração isolados com D1 e R2 locais.

## Segurança implementada

As consultas de documentos sempre incluem o setor obtido da sessão validada no servidor. O corpo da requisição não define o setor autorizado. A listagem, as alterações, o download da versão atual e das versões anteriores repetem a autorização. A API devolve 404 ao tentar acessar um documento de outro setor. As contas administrativas não possuem permissão implícita de leitura.

Nesta versão de teste, novas senhas contêm exatamente 8 números; senhas anteriores continuam válidas no login. As senhas recebem salt aleatório e PBKDF2-HMAC-SHA256 de 100.000 iterações, dentro do limite da Web Crypto de Workers. Antes do uso institucional, revisar custo de hash/algoritmo conforme infraestrutura e política de segurança. Referências: https://developers.cloudflare.com/workers/runtime-apis/web-crypto/ e https://github.com/cloudflare/workerd/issues/1346.

A sessão expira em 8 horas e utiliza cookie HttpOnly, SameSite=Strict e Secure em HTTPS. Mutações exigem Origin idêntico e token CSRF da sessão. Login recebe limite persistente por usuário e IP, com janela de 15 minutos.

Arquivos de até 10 MB: PDF, DOCX, XLSX, TXT UTF-8, PNG e JPG. A API verifica extensão e assinatura básica e limita a leitura do corpo. Essa verificação não equivale a antivírus ou inspeção profunda de documentos Office. Todos os downloads usam Content-Disposition: attachment, no-store e nosniff. O bucket não recebe URLs públicas.

## Primeiro acesso hospedado

Abra o endereço privado do sistema e crie o administrador. Cadastre os funcionários, atribua o setor e entregue as credenciais individualmente. O funcionário precisa definir sua senha pessoal no primeiro acesso. As contas de demonstração não são criadas no banco hospedado.

O ambiente está privado na plataforma Sites. Para testar o acesso com outras pessoas, definir a audiência com o responsável. A conta de aplicativo não concede acesso à camada privada da hospedagem. Não mudar automaticamente a audiência.

## Desenvolvimento e testes

1. Instale dependências com o instalador do projeto.
2. Gere migrações com `npm run db:generate` após alterar o esquema.
3. Construa o Worker com `npm run build`.
4. Aplique as migrações locais em ordem com Wrangler e `.wrangler/state`, conforme a documentação do starter.
5. Use `npm run dev` para desenvolver.
6. Execute `node tests/access-control.mjs`. O teste cria bases descartáveis e verifica autenticação, separação de setores, download, versões, CSRF, formatos, lixeira, sessões e limite de tentativas.

`scripts/seed-local-demo.mjs` cria apenas dados fictícios na base LOCAL usando Wrangler com `--local`. Nunca é executado no build, na implantação ou no banco hospedado. As credenciais presentes nesse script são exclusivamente de teste local. O banco local e o hospedado são separados.

## Limites e próxima etapa

Esta é uma versão inicial para testes com documentos fictícios. Ainda não inclui rotina de backup externo com restauração testada, antivírus, autenticação multifator, recuperação autônoma do administrador, grupos com múltiplos setores, pesquisa dentro do conteúdo dos arquivos, edição online de Word/Excel, cotas institucionais ou retenção automática. Não declara conformidade ou auditoria de segurança/LGPD.

Lixeira e versões ajudam na recuperação de erros de uso, mas não substituem backup externo. Antes de usar documentos institucionais: definir responsável de segurança, infraestrutura, controles de acesso, política de retenção, backups e recuperação, limites de armazenamento e revisão de segurança.

## Hospedagem

Este sistema precisa de um backend com D1/R2 ou de adaptação equivalente. O GitHub Pages sozinho não executa autenticação, banco de dados nem controle de documentos privados. O repositório pode guardar o código, mas a execução deve ocorrer em hospedagem compatível.

Cadastros feitos pelo formulário aguardam aprovação em Funcionários → Aprovar cadastro. No login, senha correta de conta pendente retorna uma mensagem de aprovação pendente; senha incorreta mantém a mensagem genérica. Não há liberação automática de setores.

Primeiro acesso: tanto Entrar quanto Cadastrar-se abrem o cadastro do primeiro administrador se o sistema estiver vazio. A configuração cria os setores iniciais automaticamente e, depois, o formulário público cadastra somente funcionários pendentes. Falhas de carregamento exibem Tentar novamente.

Provisionamento administrativo excepcional: ADMIN_ACCOUNT_PROVISIONING é um segredo de runtime contendo username, name, passwordHash PBKDF2 e nonce aleatório de 64 caracteres hexadecimais. Após inicialização do sistema, aplica a configuração uma única vez por nonce, preserva o ID de uma conta existente e revoga suas sessões. Registra a operação e o consumo do nonce no banco. Remover o segredo e republicar após verificar a conta; nunca incluir senhas ou a configuração em arquivos de origem. Sem o segredo, nenhuma conta é promovida.

Painel administrativo ampliado: cadastros pendentes na visão geral; edição de setor e permissões em Funcionários; documentos gerais com envio, edição de informações, download e lixeira; avisos com publicação, edição e arquivamento; relatórios acumulados por setor sem títulos ou arquivos restritos; exportação administrativa em JSON sem senhas, sessões ou arquivos. A página Backups informa que o backup automático completo não está configurado e mostra a data real da última exportação gerada. Não existe restauração automática dessa exportação.
Permissões dos setores: read permite consultar e baixar; upload acrescenta envio de novos documentos; manage permite alterar, versionar e mover/restaurar da lixeira. As permissões são verificadas no servidor. Cadastros novos pelo formulário recebem read; contas anteriores mantêm manage até revisão do administrador. Trocar setor ou permissão encerra todas as sessões do funcionário.
Documentos gerais são compartilhados com todas as contas ativas autenticadas; apenas administradores podem enviar, editar, retirar e restaurar. Documentos restritos dos setores continuam inacessíveis ao administrador. Relatórios contabilizam solicitações de download atendidas, não a conclusão no computador do usuário.
