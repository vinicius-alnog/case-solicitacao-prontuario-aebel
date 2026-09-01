# Decisões técnicas

Registro curto das escolhas de projeto e do motivo de cada uma. Cada entrada traz o que
foi decidido, o que foi descartado e o que a decisão custou — porque decisão sem custo
declarado é propaganda, não registro.

---

## 1. Três superfícies de acesso, em vez de uma API com níveis de permissão

**Contexto.** Três públicos com posturas de risco muito diferentes: um anônimo da
internet, um solicitante autenticado e um colaborador institucional.

**Descartado.** API única com um middleware genérico decidindo permissão por um campo do
token.

**Escolhido.** Três conjuntos de rotas, cada um com roteador próprio, middlewares
próprios e segredo de assinatura de sessão próprio.

**Custo.** Duplicação de middlewares e validações semelhantes em três lugares.

**Por quê mesmo assim.** A alternativa concentra a segurança inteira do sistema em um
único ponto de falha lógica. Segredos distintos por público significam que uma sessão de
solicitante não é aceita na área institucional nem por engano de código.

---

## 2. Protocolo como identificador, credencial de posse como autorização

**Contexto.** A etapa de abertura precisa aceitar edições de quem ainda não tem sessão.

**Descartado.** Usar o protocolo como credencial de edição do rascunho.

**Escolhido.** O protocolo identifica e circula — aparece no e-mail e é usado para falar
com o hospital. Uma credencial de posse separada, emitida uma única vez na criação e não
recuperável, é o que autoriza continuar editando.

**Custo.** Quem perde a credencial não retoma o rascunho por ela; precisa passar pela rota
de retomada, que tem contenções próprias.

**Por quê mesmo assim.** Se identificador e credencial fossem a mesma coisa, qualquer
pessoa que visse o protocolo poderia alterar a solicitação de outra.

---

## 3. Autenticação institucional delegada ao sistema de gestão hospitalar

**Contexto.** A equipe do SAME já tem identidade corporativa.

**Descartado.** Cadastro próprio de colaboradores no portal.

**Escolhido.** Validação da credencial no MV, com leitura do papel autorizado de lá.
Nenhuma senha institucional armazenada por este sistema.

**Custo.** Dependência: MV fora do ar significa área institucional sem autenticação.

**Por quê mesmo assim.** Um cadastro paralelo é uma segunda senha para gerenciar, um
segundo lugar para revogar acesso no desligamento e uma segunda base para vazar. E se o
MV está fora, o prontuário também está — não haveria o que responder de qualquer forma.

---

## 4. Trilha de auditoria somente-inclusão, imposta no banco

**Contexto.** O processo anterior não deixava rastro de conferência de identidade nem de
entrega.

**Descartado.** Tabela de log comum, corrigível pela aplicação.

**Escolhido.** Restrição no banco: nenhum registro pode ser alterado depois de gravado,
nem para conserto.

**Custo.** Real e já pago: quando um defeito de configuração fez a aplicação gravar a
origem errada por algumas semanas, as entradas afetadas não puderam ser corrigidas.

**Por quê mesmo assim.** Uma trilha que a aplicação consegue corrigir é uma trilha que a
aplicação consegue adulterar. Ela deixaria de provar qualquer coisa exatamente quando
mais precisasse provar.

---

## 5. Catálogos de domínio no banco, não como constantes no código

**Contexto.** Situação da solicitação, tipo de documento, tipo de solicitação e ação
auditável são conjuntos que mudam com a operação.

**Escolhido.** Tabelas de catálogo, com integridade referencial.

**Custo.** Uma consulta a mais em vários caminhos, e migrations para semear valores.

**Por quê.** Um tipo de documento novo atendido pelo SAME não deveria exigir implantação
de código. E a integridade referencial garante que nenhum registro fique com um valor que
o sistema não sabe interpretar.

---

## 6. Validação de anexo pelo conteúdo, com nome gerado pelo servidor

**Contexto.** Upload de arquivo por público anônimo é a superfície mais exposta do
sistema.

**Descartado.** Validar por extensão do nome e preservar o nome original.

**Escolhido.** Conferência do conteúdo real pelos bytes iniciais, recusa no recebimento e
nome de arquivo gerado pelo servidor. Armazenamento fora da área publicada.

**Custo.** Alguns formatos legítimos são recusados por divergência entre conteúdo e
extensão, e a mensagem de erro precisa explicar isso de forma compreensível.

**Por quê mesmo assim.** Extensão é texto que o atacante escolhe. E nome de arquivo
controlado pelo usuário é a porta de entrada de uma classe inteira de ataques por caminho.

---

## 7. Ciclo de vida por rotina agendada

**Contexto.** Três pessoas operam o serviço.

**Descartado.** Conferência manual periódica de prazos e limpeza.

**Escolhido.** Rotinas diárias para expiração de rascunho, janela de download, aviso de
vencimento e remoção definitiva — reconciliando disco contra banco.

**Custo.** Rotina agendada é código que roda sem ninguém olhando; exige registro da
própria execução para ser auditável.

**Por quê.** Prazo que depende de alguém lembrar é prazo não cumprido. Com essa equipe,
não havia margem para o contrário.

---

## 8. Frontend sem framework

**Contexto.** Duas pontas muito diferentes: celular de paciente, possivelmente em rede
ruim, e estações de trabalho institucionais. Manutenção esporádica ao longo de anos.

**Descartado.** Framework de interface com etapa de build.

**Escolhido.** HTML, CSS e JavaScript, com módulos e organização por tela.

**Custo.** Estado de tela controlado à mão, e repetição entre telas que exige disciplina
para não virar divergência.

**Por quê mesmo assim.** Sem bundle a baixar antes da primeira tela, sem etapa de build
entre corrigir e publicar, e sem dependência de interface para acompanhar. Em uma
aplicação com muito mais interatividade, a conta inverteria.

---

## 9. Contrato OpenAPI estático, sem interface navegável servida

**Contexto.** O contrato precisa ser legível por quem integra e por quem mantém.

**Descartado.** Servir uma interface de documentação interativa na própria aplicação.

**Escolhido.** Documento OpenAPI versionado junto do código, aberto em editor local.

**Custo.** Menos conveniência para quem explora a API.

**Por quê.** Evita uma dependência em tempo de execução e evita publicar o mapa completo
da API em uma rota acessível.