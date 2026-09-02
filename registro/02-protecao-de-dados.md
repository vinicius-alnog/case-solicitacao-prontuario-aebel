# Segurança, retenção e proteção de dados

Como o Portal de Solicitação de Prontuário do SAME trata dado pessoal e dado de saúde.

Documento conceitual, escrito para este estudo de caso. Descreve postura e política, não
reproduz configuração, valores de ambiente, nomes de rota ou qualquer detalhe que
facilitasse atacar o sistema real.

---

## O princípio que organiza tudo

Prontuário, selfie e documento de identidade são dados sensíveis. A regra que guiou o
projeto: **tudo o que entra precisa ter data para sair, e tudo o que acontece precisa
deixar rastro.**

O processo anterior falhava nas duas pontas. Os documentos de identidade ficavam onde
tivessem sido recebidos, sem prazo. E a entrega, feita por link de mensageiro, não
registrava quem baixou nem quando e sem processo de autenticação.

---

## Retenção: nada fica indefinidamente

Rotinas diárias aplicam os prazos automaticamente, sem depender de alguém lembrar. Os
valores abaixo são a política padrão do serviço, ajustável por ambiente.

| O que | Prazo | Contado a partir de |
|---|---|---|
| Solicitação iniciada e abandonada | 48 horas | Criação do rascunho. Retomar não estende o prazo. Depois disso, a selfie e o documento de identidade são removidos do servidor |
| Prazo do SAME para responder | 21 dias | Finalização da solicitação |
| Arquivo de resposta disponível para download | 10 dias | Data em que o SAME respondeu, com aviso por e-mail 3 dias antes do vencimento |
| Arquivo de resposta até a remoção definitiva | mais 30 dias | Fim do prazo de download |
| Documentos enviados pelo solicitante | 30 dias | Encerramento da solicitação |

**A selfie e o documento de identidade existem apenas para confirmar identidade.**
Encerrada a solicitação, são apagados junto com o resto. Não se acumulam como base de
imagens, o que seria, de longe, o ativo mais perigoso que este sistema poderia criar.

---

## Trilha de auditoria

Registra a ação, o autor, o momento e a origem. Vale tanto para a equipe do SAME quanto
para o solicitante, e **inclui tentativas de acesso que falharam**, em geral a informação
mais útil quando algo dá errado.

A restrição que a torna confiável é ser **somente-inclusão no próprio banco**: nada pode
ser alterado depois de gravado, nem para conserto. Uma trilha que a aplicação consegue
corrigir é uma trilha que a aplicação consegue adulterar.

Isso tem um preço, e ele foi pago: quando um defeito de configuração fez a aplicação
gravar a origem errada durante algumas semanas, as entradas afetadas não puderam ser
consertadas. Ficaram com o valor incorreto e uma nota explicando o artefato. O caso está
descrito em [`../analise-tecnica.md`](../analise-tecnica.md), na seção de auditoria técnica.

Acesso à trilha é restrito ao perfil administrador.

---

## Controles de acesso

**Isolamento por público.** As sessões do solicitante e do colaborador são assinadas com
segredos distintos. Uma não é aceita no lugar da outra nem por erro de programação,
porque os dois conjuntos de rotas nem sequer consultam o mesmo segredo.

**Verificação na camada de dados, não na tela.** O vínculo entre quem pede e o que é
pedido é reconferido a cada consulta. Esconder um botão não é controle de acesso.

**Nenhuma senha institucional armazenada.** O colaborador é validado no sistema de gestão
hospitalar. Quando alguém é desligado, o acesso ao portal termina junto, sem depender de
revogação em um segundo lugar.

**Troca de senha encerra sessões antigas.** Se alguém tinha acesso indevido a uma sessão
aberta, a troca corta esse acesso.

**Tentativas repetidas são contidas e registradas**, nos dois públicos.

---

## Proteção da superfície pública

A abertura da solicitação não pode exigir login, o solicitante ainda não tem cadastro. O
que a protege:

- **Limite de requisições por origem**, com tetos separados por tipo de operação.
- **Protocolo imprevisível**, que identifica a solicitação mas não autoriza nada sozinho.
- **Credencial de posse do rascunho**, emitida uma única vez na criação e não recuperável.
  É ela, e não o protocolo, que autoriza continuar editando.

**Recuperação de acesso não revela se um CPF existe na base.** A resposta é sempre a
mesma, exista ou não solicitação para aquele CPF. O contrário permitiria descobrir quem
tem prontuário no hospital testando CPFs que é, em si, uma revelação de dado de saúde.

Há uma exceção deliberada e documentada: a rota de retomada de rascunho por CPF
**precisa** confirmar que existe um rascunho em andamento, porque essa é a função dela. O
risco foi aceito com contenções explícitas, janela curta de validade do rascunho, teto
baixo de requisições por origem, toda tentativa auditada, e nenhum conteúdo de arquivo na
resposta.

Registrar um risco aceito, com o motivo e as contenções, é parte do trabalho. Risco não
registrado é risco esquecido.

---

## Proteção dos arquivos

- **Validação pelo conteúdo real do arquivo**, lido nos bytes iniciais, e não pela
  extensão do nome. Arquivo fora das regras é recusado no recebimento e não chega a ser
  guardado.
- **Nome de arquivo gerado pelo servidor.** Nada do que o solicitante digita chega ao
  sistema de arquivos, o que elimina a classe inteira de ataques por caminho.
- **Armazenamento fora da área publicada.** Não existe URL que sirva o arquivo
  diretamente: todo acesso passa por uma rota que confere a quem o arquivo pertence.
- **Registro de cada download**, com autor e momento, exatamente o que a entrega por link
  de mensageiro não produzia.

---

## Postura de configuração

O boot da aplicação **recusa subir com configuração pela metade**: se os segredos de
assinatura de sessão não passam na validação, o processo aborta com mensagem clara, em vez
de subir em estado degradado.

A lição aprendida na revisão pré-implantação foi estender esse rigor: **ausência de valor
em uma configuração de segurança deve ser tratada como erro, nunca como padrão
silencioso.** Um limitador que não limita e uma auditoria que grava sempre a mesma origem
não quebram nenhum teste, apenas param de proteger.

---

## O que não está em operação

Registrado para não gerar expectativa equivocada:

- **Cobrança pelo serviço** - a etapa existe no fluxo e está desligada; nenhuma
  solicitação é retida por falta de pagamento.
- **Leitura automática de documentos (OCR)** - prevista, não implementada. A conferência
  documental é integralmente humana.