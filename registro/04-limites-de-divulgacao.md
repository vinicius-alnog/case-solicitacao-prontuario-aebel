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
- **Capturas de tela do sistema em operação.**

---

## O que é sintético

- **Todos os diagramas** em `midia/` foram desenhados especificamente para este
  repositório. São representações conceituais: não reproduzem o desenho técnico interno,
  o modelo físico do banco nem a topologia real.
- **Todas as telas** em `midia/interfaces/` foram recriadas. Reproduzem a estrutura e a
  identidade visual das interfaces, com conteúdo inteiramente inventado — protocolos,
  iniciais, datas, contadores e nomes de arquivo são fictícios.
- **Os exemplos** em `amostras/` são inventados. Ilustram formato e vocabulário do
  domínio, não payloads reais e não o contrato real da API.

Onde há número de operação — 200 solicitações atendidas, tempo médio de 7 dias corridos,
3 usuários institucionais — trata-se de indicador agregado, sem qualquer possibilidade de
identificação individual.

---

## Sobre os números do "antes"

O tempo médio de aproximadamente 20 dias atribuído ao processo anterior é uma referência
institucional, não uma medição instrumentada — a ausência de indicadores era parte do
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