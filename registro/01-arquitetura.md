# Arquitetura

Detalhamento das camadas e integrações do Portal de Solicitação de Prontuário do SAME —
Hospital Evangélico de Londrina (AEBEL).

Documento conceitual. Não reproduz código, nomes de rota, esquema real de banco,
topologia de rede, nomes de servidor ou variáveis de ambiente do sistema em operação.

---

## Visão em camadas

O backend é uma aplicação Node.js com Express, sobre banco Oracle acessado por um
construtor de consultas. As responsabilidades estão separadas em quatro camadas, e a
regra que sustenta a separação é simples: **cada camada só conhece a camada imediatamente
abaixo**.

| Camada | Responsabilidade | O que ela nunca faz |
|---|---|---|
| Roteadores | Definem as rotas de uma superfície de acesso e montam os middlewares dela | Não contêm regra de negócio |
| Controllers | Validam a requisição, orquestram os serviços e montam a resposta | Não acessam o banco diretamente |
| Services | Acesso a dados, integrações externas e regra de negócio | Não conhecem objetos de requisição ou resposta |
| Helpers | Sessão, upload, limite por origem, agendamento e utilitários | Não decidem regra de negócio |

O ganho concreto dessa disciplina aparece nos testes: um serviço pode ser exercitado sem
subir servidor HTTP, e uma rota pode ser exercitada com o middleware real no caminho —
que é o único jeito de testar de verdade o comportamento de upload e de sessão.

---

## As três superfícies de acesso

O sistema não é uma API única com níveis de permissão. São três conjuntos de rotas, cada
um com seu roteador e sua postura de segurança.

### Abertura da solicitação — pública

Aceita requisições de quem ainda não tem cadastro, porque não há alternativa: o
solicitante chega ao sistema sem nunca ter passado por ele.

O que protege esta superfície:

- **Limite de requisições por origem** em todas as rotas, com tetos separados por tipo de
  operação — criar, enviar anexo, editar rascunho, retomar, ler anexo.
- **Credencial de posse do rascunho**, emitida uma única vez no momento da criação. É ela
  que autoriza continuar editando aquela solicitação.
- **Protocolo imprevisível**, que identifica a solicitação, mas não autoriza nada por si.

### Portal do solicitante — sessão própria

Credenciais geradas ao final da abertura e enviadas por e-mail, com troca obrigatória no
primeiro acesso. A sessão é assinada com um segredo exclusivo deste público.

Cada consulta reconfere o vínculo entre a sessão e a solicitação pedida — a verificação
está na camada de dados, não apenas na tela. Uma tela escondida não é um controle de
acesso.

### Área institucional — identidade do MV

O colaborador se autentica com a mesma matrícula que já usa no sistema de gestão
hospitalar. A aplicação valida a credencial lá, lê o papel autorizado e emite uma sessão
institucional assinada com um segredo **distinto** do usado para solicitantes.

Consequência dessa separação: uma sessão de solicitante não é aceita na área
institucional nem por erro de programação, porque as rotas institucionais sequer
consultam o segredo do outro público.

---

## Ciclo de uma requisição

1. **Entrada.** A aplicação lê o endereço de origem real da requisição, contando os saltos
   de proxy reverso configurados. Esse endereço alimenta o limitador e a trilha de
   auditoria — os dois dependem dele estar correto.
2. **Limite por origem.** Rotas públicas passam por um limitador com teto próprio.
3. **Sessão.** Nas superfícies autenticadas, a sessão é verificada contra o segredo daquele
   público específico.
4. **Controller.** Valida o corpo da requisição e orquestra os serviços.
5. **Service.** Aplica a regra, acessa o banco com consulta parametrizada e integra com o
   que for necessário.
6. **Auditoria.** Ações relevantes gravam uma linha somente-inclusão com ação, autor,
   momento e origem.
7. **Resposta.** Erros passam por um tratamento central, que devolve mensagem útil sem
   revelar detalhe interno.

---

## Integrações

| Integração | Papel | Postura adotada |
|---|---|---|
| Oracle | Solicitações, catálogos, auditoria e metadados de arquivo | Consultas sempre parametrizadas; nenhuma concatenação de SQL |
| MV | Valida o colaborador e devolve o papel autorizado | Nenhuma senha institucional é armazenada por este sistema |
| SMTP institucional | Credenciais, andamento, resposta e aviso de vencimento | Templates versionados no banco; envio registrado |
| Armazenamento em disco | Anexos do solicitante e arquivos de resposta | Fora da área publicada; nome gerado pelo servidor; todo acesso passa por rota que confere titularidade |
| Canal de tempo real | Sinaliza a solicitação já aberta por outro colaborador | Complementar: a fila funciona sem ele, apenas com mais risco de trabalho duplicado |

---

## Rotinas agendadas

Executam diariamente, sem intervenção humana, e registram a própria execução:

- expiração de rascunho abandonado, com remoção dos anexos de identidade;
- encerramento da janela de download do arquivo respondido;
- aviso por e-mail antes do vencimento do prazo de download;
- remoção definitiva dos arquivos vencidos.

A rotina de remoção **reconcilia disco contra banco**: verifica os dois lados em vez de
confiar apenas no registro. Um arquivo órfão no disco é tão problema de retenção quanto
uma linha órfã na tabela.

---

## Interface

18 telas em HTML, CSS e JavaScript, sem framework, organizadas em três conjuntos: fluxo
público em etapas, portal do solicitante e painel administrativo.

Seguem a identidade visual institucional — azul institucional como autoridade, turquesa
como ação positiva, Arial, superfícies claras e cantos amplos. No celular, as tabelas
administrativas se reorganizam em cards em vez de comprimir colunas, e a ação principal
fica ao alcance do polegar, no fim do fluxo.

O motivo de não haver framework está registrado em
[`03-escolhas-e-custos.md`](03-escolhas-e-custos.md).

---

## Contrato da API

O contrato das rotas é mantido como documento OpenAPI versionado junto do código. A
aplicação **não serve interface de documentação navegável**: isso evita uma dependência em
tempo de execução e evita publicar o mapa completo da API em uma rota acessível.