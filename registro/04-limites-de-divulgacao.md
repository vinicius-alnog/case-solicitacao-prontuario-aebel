# Confidencialidade

Este repositório documenta um sistema corporativo em operação no Hospital Evangélico de
Londrina (AEBEL). Foi montado deliberadamente para contar o projeto **sem** comprometer o
sistema, a instituição ou qualquer paciente.

---

## O que não está aqui, e não estará

- **Código-fonte**, de qualquer camada, em qualquer trecho.
- **Esquema real do banco**: nomes de tabela, colunas, chaves, índices ou tipos.
- **Nomes de rota, parâmetros ou contrato da API.**
- **Variáveis de ambiente, segredos, credenciais, chaves ou strings de conexão.**
- **Topologia de rede, nomes de servidor, endereços, portas ou configuração de
  infraestrutura.**
- **Dados de paciente**, em qualquer forma: nome, CPF, protocolo real, documento,
  prontuário, e-mail ou registro de atendimento.
- **As telas restritas**, em qualquer forma: a fila de trabalho do SAME, o painel
  administrativo e qualquer superfície que exija credencial institucional. Nem captura,
  nem mock fiel, nem reprodução do layout.
- **Capturas de produção**, de qualquer superfície, inclusive da pública.

---

## O que é sintético

- **Todos os diagramas** em `midia/` foram desenhados especificamente para este
  repositório. São representações conceituais: não reproduzem o desenho técnico interno,
  o modelo físico do banco nem a topologia real.
- **As telas do fluxo público** em `midia/interfaces/publico/` são capturas reais, feitas
  no ambiente de **homologação**, nunca em produção. O fluxo de solicitação é aberto a
  qualquer pessoa, então a tela em si não é informação reservada; o que foi controlado é o
  conteúdo. Nome, CPF, e-mail, vínculo, protocolo e usuário de acompanhamento são
  inventados, e os arquivos anexados são imagens sintéticas, não documentos de identidade.
  O CPF usado tem dígito verificador válido e nenhum paciente correspondente na base.
- **As telas restritas não aparecem, nem recriadas.** Houve uma versão anterior deste
  repositório com um mock fiel da fila do SAME, e ela foi retirada: reproduzir com
  precisão uma tela que só abre com credencial não acrescentava nada ao relato e servia de
  molde para uma página falsa apontada à própria equipe. No lugar entrou
  [`midia/ciclo-de-vida.svg`](../midia/ciclo-de-vida.svg), que descreve os estados da
  solicitação e as contagens derivadas deles, sem desenhar interface.
- **Os exemplos** em `amostras/` são inventados. Ilustram formato e vocabulário do
  domínio, não payloads reais e não o contrato real da API.

Onde há número de operação: 200 solicitações atendidas, tempo médio de 7 dias corridos,
3 usuários institucionais, trata-se de indicador agregado, sem qualquer possibilidade de
identificação individual.

---

## Sobre os números do "antes"

O tempo médio de aproximadamente 20 dias atribuído ao processo anterior é uma referência
institucional, não uma medição instrumentada, a ausência de indicadores era parte do
problema que o sistema veio resolver. Essa ressalva aparece junto de cada citação do
número, no [README](../README.md) e no [case study](../ANALISE-TECNICA.md), para que ele não
seja lido como resultado obtido sob medição controlada.

---

## Propriedade intelectual

O sistema descrito aqui foi desenvolvido no âmbito do vínculo profissional do autor com a
AEBEL, e **a propriedade intelectual é da instituição**. Este repositório é um relato de
experiência profissional: descreve o problema enfrentado, as decisões tomadas e o
resultado observado.

Não constitui, e não deve ser lido como, autorização de uso, licença, cessão ou
distribuição de qualquer parte do sistema.