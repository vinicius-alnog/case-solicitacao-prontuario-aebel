# Case study - Portal de Solicitação de Prontuário do SAME

**Hospital Evangélico de Londrina - AEBEL**

| Campo | Informação |
|---|---|
| Sistema | Portal de solicitação de documentos sob responsabilidade do SAME |
| Papel | Desenvolvedor responsável: modelagem, backend, frontend, integração e implantação |
| Início | Especificação funcional em 14/07/2026; primeiro commit em 24/07/2026 |
| Produção | Agosto de 2026 |
| Escala construída | 36 rotas, 15 tabelas, 16 migrations, 18 telas, 42 arquivos de teste |
| Documento | Aprofundamento técnico do [README](README.md) |

Este documento explica **por que** o sistema ficou como ficou. Ele não reproduz código,
esquema real de banco, nomes de rota, variáveis de ambiente ou qualquer configuração do
ambiente em operação.

---

## 1. O ponto de partida

O SAME é o serviço que guarda o prontuário do paciente. Quando alguém precisa de uma
cópia para perícia do INSS, tratamento em outro serviço, processo judicial, o pedido
chegava presencialmente ou pelo SAC.

A partir daí, tudo acontecia de um lado só do balcão. Havia um sistema institucional ao
qual apenas o colaborador do SAME tinha acesso: ele localizava o prontuário, anexava o
arquivo e mandava um link pelo WhatsApp para o solicitante baixar.

Esse desenho tinha um problema mais sério do que a lentidão. pois **ele não deixava rastro.**
Não havia registro estruturado de quem conferiu a identidade do solicitante, com base em
qual documento, em que data, nem de quando e para quem a cópia foi entregue. Também não
havia prazo nem para o hospital responder, nem para o arquivo sair do ar depois de
entregue, nem para os documentos de identidade enviados pelo solicitante serem
descartados.

Para um documento de saúde, isso é exposição do paciente e da instituição.

---

## 2. As restrições que moldaram o projeto

Quatro restrições apareceram antes de qualquer linha de código, e todas as decisões
seguintes derivam delas.

**O público externo não tem cadastro e não pode ter.** Quem pede prontuário pode nunca
ter usado um sistema do hospital. Exigir cadastro prévio simplesmente devolveria a pessoa
ao balcão. O sistema precisava aceitar um desconhecido, e ainda assim ter certeza
suficiente de quem ele é.

**A equipe interna já tem identidade.** Os colaboradores do SAME são usuários do sistema
de gestão hospitalar (MV). Criar um cadastro paralelo significaria uma segunda senha para
gerenciar, um segundo lugar para revogar acesso quando alguém sai, e uma segunda base
para vazar.

**A operação é pequena.** Três pessoas. O sistema não podia depender de disciplina manual
para funcionar: se um prazo dependesse de alguém lembrar de conferir uma planilha, ele
não seria cumprido.

**O dado é sensível por definição.** Prontuário, selfie, documento de identidade. Tudo o
que entra tem que ter data para sair.

---

## 3. Decisões de arquitetura

### 3.1. Três superfícies de acesso, não uma API com níveis de permissão

A decisão estrutural do projeto foi separar a aplicação em **três conjuntos de rotas com
posturas de segurança distintas**, cada um com seu próprio roteador, em vez de uma API
única resolvendo autorização caso a caso.

| Superfície | Quem acessa | Como é protegida |
|---|---|---|
| Abertura da solicitação | Público, sem cadastro | Nenhuma sessão. Limite de requisições por origem em todas as rotas e uma credencial de posse do rascunho, emitida uma única vez na criação |
| Portal do solicitante | Paciente ou representante legal | Sessão própria, emitida ao final da abertura, com segredo de assinatura exclusivo desse público |
| Área institucional | Colaborador e administrador do SAME | Sessão institucional, com segredo de assinatura **distinto** do anterior, emitida só após validação no MV |

O ganho não é organizacional, é de contenção. Segredos de assinatura separados por
público significam que uma sessão de solicitante não é aceita como sessão institucional,
nem por engano de código: os dois conjuntos de rotas nem sequer consultam o mesmo
segredo. E o boot da aplicação recusa subir se esses segredos não passarem na validação,
não existe o estado intermediário de "no ar com configuração pela metade".

O custo é duplicação: middlewares e validações parecidas em três lugares. Foi um custo
aceito conscientemente, porque a alternativa que seria um middleware genérico decidindo permissão
por um campo do token, colocaria a segurança inteira do sistema em um único ponto de
falha lógica.

### 3.2. O protocolo é identificador, não credencial

A etapa de abertura precisa aceitar requisições de quem ainda não tem sessão. A pergunta
é: o que autoriza a pessoa a continuar editando aquele rascunho?

A resposta foi separar as duas coisas. O **protocolo** é o número que identifica a
solicitação do começo ao fim, aparece no e-mail, o solicitante usa para falar com o
hospital, e por isso circula. O que autoriza a edição do rascunho é uma **credencial de
posse emitida uma única vez na criação**, que não circula e não é recuperável.

Se as duas coisas fossem a mesma, qualquer pessoa com o número do protocolo, inclusive
quem visse o e-mail por cima do ombro, poderia alterar a solicitação de outra.

### 3.3. Autenticação institucional delegada ao MV

O colaborador entra com a mesma matrícula que já usa no sistema de gestão hospitalar. O
portal valida a credencial lá e lê dali o papel que autoriza o acesso: colaborador ou
administrador. **Nenhuma senha institucional é armazenada por este sistema.**

O efeito prático aparece no desligamento de um funcionário: o acesso ao portal termina
junto com o acesso ao MV, sem depender de alguém lembrar de revogar em um segundo lugar.

O trade-off é a dependência: se o MV estiver fora, a área institucional não autentica. Foi
aceito porque, se o MV está fora, o prontuário também está, a equipe do SAME não teria o
que responder de qualquer forma.

### 3.4. Trilha de auditoria somente-inclusão, imposta no banco

A auditoria registra a ação, o autor, o momento e a origem, e inclui **tentativas de
acesso que falharam**, que costumam ser a informação mais útil quando algo dá errado.

A restrição que a torna útil é ela ser somente-inclusão no próprio banco: **nada pode ser
alterado depois de gravado, nem para conserto.** Uma trilha que a aplicação consegue
corrigir é uma trilha que a aplicação consegue adulterar, e aí ela não prova nada.

Isso tem uma consequência incômoda que vale registrar honestamente: quando um defeito de
configuração fez a aplicação gravar a origem errada por algumas semanas (ver §6), as
entradas antigas **não puderam ser consertadas**. Ficaram com o valor errado e uma nota
explicando o artefato. Esse é exatamente o preço da garantia e pagá-lo foi a decisão
certa.

### 3.5. Validação de anexo pelo conteúdo, não pela extensão

Todo anexo é conferido no recebimento: formato, tamanho e se o conteúdo real do arquivo
corresponde ao tipo que ele diz ser, lido pelos bytes iniciais. Um arquivo que falha em
qualquer dessas checagens é recusado na hora e não chega a ser guardado.

Além disso, **o nome com que o arquivo é gravado é gerado pelo servidor**. Nada do que o
solicitante digita chega ao sistema de arquivos, o que elimina de uma vez a classe inteira
de ataques por caminho de arquivo. E o diretório de armazenamento fica fora da área
publicada na internet: não existe URL que sirva o arquivo diretamente, todo acesso passa
por uma rota que confere a quem aquele arquivo pertence.

### 3.6. Ciclo de vida por rotina agendada, não por disciplina humana

Com três pessoas na operação, qualquer prazo que dependesse de alguém lembrar seria um
prazo não cumprido. Rotinas diárias aplicam, sem intervenção:

- expiração do rascunho abandonado, com remoção dos documentos de identidade já enviados;
- fim da janela de download do arquivo respondido;
- aviso por e-mail antes de o prazo de download vencer;
- remoção definitiva dos arquivos quando o prazo termina.

A rotina de remoção **reconcilia disco contra banco** em vez de confiar apenas no
registro: um arquivo órfão no disco é tão problema de retenção quanto uma linha órfã na
tabela.

### 3.7. Frontend sem framework

As 18 telas foram construídas em HTML, CSS e JavaScript, sem framework de interface.

A escolha não foi ideológica. O portal é acessado de duas pontas muito diferentes: o
celular de um paciente, que pode estar em rede ruim, e as estações de trabalho da
instituição. Sem framework, não há bundle a baixar antes da primeira tela aparecer, não há
etapa de build entre corrigir e publicar, e não há atualização de dependência de interface
para acompanhar em um sistema que precisa durar anos com manutenção esporádica.

O custo é real e aparece na manutenção: estado de tela é controlado à mão, e a
repetição entre telas precisa de disciplina para não virar divergência. Em uma aplicação
com muito mais interatividade, a conta inverteria.

### 3.8. Contrato OpenAPI estático, sem interface de documentação servida

O contrato das rotas é mantido como documento OpenAPI versionado junto do código, mas a
aplicação **não serve uma interface de documentação navegável**. Isso evita adicionar
dependência em tempo de execução e, principalmente, evita publicar o mapa completo da API
em uma rota acessível.

Quem precisa navegar o contrato abre o arquivo em um editor de OpenAPI local.

---

## 4. Modelo de dados

Quinze tabelas, evoluídas em 16 migrations numeradas e versionadas junto do código. A
organização segue três grupos:

**O núcleo do processo.** A solicitação é o registro central: protocolo, tipo de
solicitante, situação atual, prazos e vínculo com o CPF informado. Em volta dela ficam os
anexos enviados, o arquivo de resposta e o registro de cada download.

**Os catálogos de domínio.** Situação da solicitação, tipo de solicitação, tipo de
documento e ação auditável são tabelas, não constantes no código. O motivo é operacional:
uma situação nova, ou um novo tipo de documento atendido pelo SAME, não deveria exigir
implantação de código e a integridade referencial garante que nenhum registro fique com
um valor que o sistema não sabe interpretar.

**O registro do que aconteceu.** A trilha de auditoria e o histórico de e-mails enviados,
ambos somente-inclusão, com os templates de e-mail versionados no próprio banco para que
mudar um texto institucional não seja mudar código.

---

## 5. A jornada, do lado de quem usa

**Etapas curtas por decisão de produto.** O público inclui pessoas com pouca familiaridade
com sistemas digitais. Um formulário longo em página única produz abandono; etapas curtas
com progresso visível produzem conclusão.

**A conferência final devolve tudo de uma vez.** Ao finalizar, o sistema verifica a
solicitação inteira e informa **todas** as pendências em uma única resposta, em vez de
revelar uma por vez a cada tentativa. Descobrir um problema por vez é a forma mais
confiável de fazer alguém desistir.

**Recusar documento é diferente de cancelar solicitação.** São ações distintas com efeitos
distintos, e essa separação foi deliberada:

- **Recusa** devolve a solicitação ao solicitante indicando exatamente o que reenviar e
  por quê. Ele reenvia apenas o item recusado, pelo portal, e a solicitação volta sozinha
  para a fila quando não resta pendência, sem ninguém precisar recolocá-la lá.
- **Cancelamento** encerra o caso com justificativa registrada, quando a documentação não
  permite seguir.

**Uma solicitação em andamento por CPF.** Se a pessoa começa uma nova sem concluir a
anterior, a anterior é encerrada automaticamente e isso fica registrado. A alternativa (permitir várias) 
produziria uma fila de rascunhos duplicados que a equipe teria que desempatar manualmente.

**Sinalização em tempo real na fila.** Um canal de tempo real marca a solicitação que já
está aberta por outro colaborador. Com três pessoas trabalhando na mesma fila, o problema
não é hipotético: sem isso, duas pessoas conferem os mesmos anexos e uma das duas jogou o
trabalho fora.

---

## 6. Auditoria técnica antes da implantação

Antes de colocar em produção, o código passou por uma revisão sistemática de bugs, casos
de borda e segurança, com cada achado apontando arquivo e linha, e separando o que foi
verificado por execução do que foi verificado por leitura.

O achado de maior impacto foi também o mais barato de corrigir, e é instrutivo porque
**nenhum teste falhava por causa dele**: a aplicação estava atrás de um proxy reverso, mas
sem a configuração que a instrui a confiar nele para determinar o endereço de origem da
requisição. Com isso, toda requisição parecia vir do próprio servidor.

Dois efeitos, ambos silenciosos:

- **A auditoria perdia a origem.** Toda linha gravava o mesmo endereço, prometendo uma
  rastreabilidade que não estava sendo cumprida.
- **Os limitadores por origem viravam um balde único.** Como o limite é contado por
  endereço de origem, e todas as requisições apresentavam o mesmo, o contador passou a ser
  global. Na prática: alguém esgotaria o limite de tentativas de login e **ninguém mais no
  hospital conseguiria entrar** pelo período seguinte, negação de serviço trivial, e ao
  mesmo tempo nenhuma contenção real contra quem estivesse atacando.

Vale registrar a assimetria que escondeu o defeito: o código já avisava quando essa
configuração estava permissiva demais, e ficava calado quando estava ausente, que era o
caso perigoso. O alerta apontava para o lado errado.

A revisão encontrou outras assimetrias do mesmo tipo: uma proteção presente em um lugar e
ausente no lugar equivalente. É o padrão de defeito de código cuidado, e é justamente o
que uma revisão sistemática encontra e uma leitura casual não.

---

## 7. Qualidade

- **42 arquivos de teste automatizado**, divididos entre teste de rota, com o middleware
  real no caminho, e teste de unidade de serviços e utilitários.
- **Os testes de upload exercitam o middleware de verdade**, gravando arquivo em disco.
  Testar upload contra um middleware simulado testa a simulação, não o comportamento e
  o comportamento é justamente onde estão as decisões de segurança.
- **Roteiros de fumaça separados da suíte**, para verificar integrações reais sob comando
  explícito, sem tornar a suíte dependente de serviço externo.
- **Migrations numeradas e versionadas** junto do código, de modo que o estado do banco
  seja reconstituível e revisável.

---

## 8. Resultados

| Indicador | Antes | Depois |
|---|---|---|
| Tempo médio até a entrega | ~20 dias | 7 dias corridos |
| Canal de entrega | Link por WhatsApp | Download pelo portal, com prazo e registro |
| Rastreabilidade | Inexistente | Trilha somente-inclusão de ponta a ponta |
| Retenção de documentos | Sem prazo definido | Prazos aplicados por rotina diária |
| Indicadores de operação | Inexistentes | Painel com fila, pendências e prazos |
| Solicitações atendidas | - | 200 desde agosto de 2026 |
| Equipe de operação | 3 pessoas | 3 pessoas |

**Ressalva metodológica.** Os 20 dias do processo anterior são referência institucional, e
não medição instrumentada, a ausência de indicadores era parte do problema que o sistema
veio resolver. Os 7 dias são apurados sobre os registros do sistema atual. A comparação
indica ordem de grandeza, não resultado obtido sob medição controlada.

---

## 9. O que ficou de fora, deliberadamente

- **Cobrança pelo serviço.** A etapa existe no fluxo e está desligada. Enquanto estiver,
  nenhuma solicitação é retida por falta de pagamento. Deixar a etapa estruturada e
  dormente custa pouco; refazer o fluxo depois para encaixá-la custaria muito.
- **Leitura automática de documentos (OCR).** Prevista, não implementada. A conferência
  documental é integralmente humana, o que, para validação de identidade, é a postura
  conservadora correta.
- **Retomada de solicitação após a expiração da sessão.** O rascunho abandonado vence e
  seus anexos são removidos. Reter documento de identidade por mais tempo para conveniência
  de retomada é um mau negócio de proteção de dados.

---

## 10. O que eu faria diferente

**Instrumentar o processo antigo antes de substituí-lo.** A melhor comparação que posso
oferecer hoje tem uma ressalva metodológica, e ela é evitável: bastaria ter medido o
processo manual por algumas semanas antes de trocá-lo.

**Fazer o boot recusar configuração ambígua, não só configuração ausente.** O sistema já
recusa subir sem os segredos de sessão. Se aplicasse o mesmo rigor à configuração de
proxy reverso, o defeito descrito na §6 teria aparecido na primeira subida do ambiente,
não em uma revisão semanas depois. A lição é generalizável: **ausência de valor em uma
configuração de segurança deve ser tratada como erro, nunca como padrão silencioso.**

**Investir mais cedo em teste de integração da conferência documental.** É o ponto do
fluxo com mais estados possíveis: aprovado, recusado item a item, reenviado
parcialmente, cancelado e foi onde o teste manual mais custou tempo.