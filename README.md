# Orquestração com AWS Step Functions – Workflow de Validação de Arquivos

## O que é o AWS Step Functions?  
O **AWS Step Functions** é um serviço de **orquestração de fluxos de trabalho** da AWS que permite coordenar múltiplos serviços de forma **visual, automatizada e com pouco código**.  
Ele utiliza o conceito de **máquina de estados**, onde cada etapa do processo é representada por um **state** (estado), que pode ser uma função Lambda, uma chamada de serviço AWS, uma decisão condicional ou até mesmo loops.  

### Benefícios do Step Functions  
- **Automação** de processos complexos com mínima intervenção manual.  
- **Integração nativa** com serviços como **Lambda, S3, DynamoDB, SNS, SQS, ECS, Glue** e muitos outros.  
- **Monitoração e logs detalhados** no próprio console da AWS.  
- **Tolerância a falhas**, permitindo **retries, timeouts** e tratamento de erros de forma centralizada.  
- **Execução visual**, facilitando o entendimento e a manutenção dos fluxos.  

---

## Workflow de Validação de Arquivos  

Este diagrama representa um **pipeline de validação de arquivos** automatizado com o Step Functions. Ele garante que cada arquivo enviado passe por verificações antes de ser aceito ou rejeitado, registrando logs e notificando os responsáveis. 

<img width="527" height="503" alt="image" src="https://github.com/user-attachments/assets/38519817-ab1d-4708-b2e8-a6f75b6d5324" />


### Fluxo detalhado  

1. **Start**  
   - O workflow é iniciado quando um arquivo é enviado para o sistema (ex.: upload em bucket S3, acionando um evento).  

2. **Lambda: VALIDAMETADADOS**  
   - Primeira função Lambda responsável por validar **metadados do arquivo** (ex.: extensão, tamanho, nome, permissões).  

3. **Lambda: VALIDARCONTEUDO**  
   - Segunda função Lambda que realiza a **validação do conteúdo do arquivo** (ex.: verificar se o JSON está bem formatado, se o CSV segue o padrão esperado ou se o documento não contém erros estruturais).  

4. **Choice State: ESCOLHA**  
   - Aqui o Step Functions decide o próximo caminho do workflow:  
     - Se o arquivo for **válido** → segue para a rota de sucesso.  
     - Se o arquivo for **inválido** → segue para a rota de erro.  

5. **Rota de Sucesso (SUCCESS)**  
   - **DynamoDB: GRAVAR NO DynamoDB** → Os dados do arquivo validado são gravados no banco para registro e auditoria.  
   - **SNS: NOTIFICAR SUCESSO** → Um tópico SNS envia notificações (e-mail, SMS, integração com sistemas externos) informando que o arquivo foi processado com êxito.  
   - **Succeed State: ÊXITO** → Finaliza o workflow com status de sucesso.  

6. **Rota de Erro (ERROR)**  
   - **SQS: ENVIAR PARA SQS** → Caso o arquivo seja inválido, a mensagem é enviada para uma fila no SQS, permitindo análise posterior ou reprocessamento.  
   - **SNS: NOTIFICAR ERRO** → Um tópico SNS notifica a equipe responsável de que houve uma falha na validação.  
   - **Fail State: FALHA** → Finaliza o workflow com status de erro.  

---

## Vantagens desse Workflow  
- **Automatização completa** da validação de arquivos.  
- **Registro de logs estruturados** no DynamoDB.  
- **Notificações automáticas** em casos de sucesso ou erro.  
- **Tratamento robusto de falhas** usando filas (SQS).  
- **Escalabilidade** nativa da AWS (Lambda, SQS e SNS se adaptam à demanda).  

---

## Exemplo de Definição do Step Functions (JSON Simplificado)  

```json
{
  "Comment": "Workflow de Validação de Arquivos",
  "StartAt": "ValidarMetadados",
  "States": {
    "ValidarMetadados": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:ValidaMetadados",
      "Next": "ValidarConteudo"
    },
    "ValidarConteudo": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:ValidaConteudo",
      "Next": "DecisaoArquivo"
    },
    "DecisaoArquivo": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.valido",
          "BooleanEquals": true,
          "Next": "GravarNoDynamoDB"
        }
      ],
      "Default": "EnviarParaSQS"
    },
    "GravarNoDynamoDB": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:SalvarDynamoDB",
      "Next": "NotificarSucesso"
    },
    "EnviarParaSQS": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:EnviarErroSQS",
      "Next": "NotificarErro"
    },
    "NotificarSucesso": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:NotificaSNS",
      "End": true
    },
    "NotificarErro": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:NotificaErroSNS",
      "End": true
    }
  }
}
