#  Assistente de Delivery Inteligente com AWS Step Functions e Amazon Bedrock

Projeto prático desenvolvido para o bootcamp da **DIO** demonstrando a orquestração de um fluxo de atendimento e pedidos de delivery automatizado, combinando a inteligência generativa do **Amazon Bedrock** com a orquestração serverless do **AWS Step Functions**.

---

##  Objetivo do Projeto

Criar uma arquitetura de microsserviços serverless capaz de atender clientes de forma humanizada, sugerir pratos personalizados via Inteligência Artificial e coordenar com segurança todas as etapas de um pedido de delivery (validação, pagamento, envio para a cozinha e entrega).

---

##  Arquitetura da Solução

O fluxo do pedido é orquestrado como uma máquina de estados no **AWS Step Functions**, garantindo que cada etapa só avance após a conclusão bem-sucedida da anterior:

1. **Recepção da Mensagem:** O cliente interage solicitando recomendações ou fazendo um pedido.
2. **Consulta ao Amazon Bedrock:** A IA analisa as preferências do cliente e o cardápio disponível para gerar recomendações personalizadas em linguagem natural.
3. **Validação do Pedido:** O Step Functions verifica a disponibilidade dos itens no restaurante.
4. **Processamento de Pagamento:** Execução da cobrança via cartão ou PIX com tratamento de falhas e retentativas automáticas.
5. **Notificação e Preparo:** O restaurante recebe o pedido na cozinha e o cliente é notificado do status em tempo real via **Amazon SNS**.
6. **Despacho para Entrega:** Acionamento do entregador e finalização do ciclo do pedido.

---

##  Tecnologias Utilizadas

- **AWS Step Functions:** Orquestrador visual de fluxos de trabalho (máquina de estados serverless).
- **Amazon Bedrock:** Serviço gerenciado para modelos de fundação de IA generativa (usado para o assistente conversacional e recomendações).
- **AWS Lambda:** Funções que executam as regras de negócio individuais (calcular frete, cobrar pagamento, enviar para a cozinha).
- **Amazon DynamoDB:** Banco de dados NoSQL para registrar o histórico dos pedidos e status dos clientes.
- **Amazon SNS:** Serviço de notificações para avisar o cliente a cada mudança de status do pedido.

---

##  Definição da Máquina de Estados (Step Functions JSON)

Abaixo está o modelo em formato Amazon States Language (ASL) que define o fluxo de trabalho do assistente de delivery:

```json
{
  "Comment": "Fluxo de Orquestracao do Assistente de Delivery com Bedrock",
  "StartAt": "GerarRecomendacaoComBedrock",
  "States": {
    "GerarRecomendacaoComBedrock": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "anthropic.claude-3-sonnet-20240229-v1:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 300,
          "messages": [
            {
              "role": "user",
              "content": "Sugira itens do cardapio com base nas preferencias do cliente."
            }
          ]
        }
      },
      "Next": "ClienteConfirmouPedido"
    },
    "ClienteConfirmouPedido": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pedidoConfirmado",
          "BooleanEquals": true,
          "Next": "ProcessarPagamento"
        }
      ],
      "Default": "AguardarInteracaoCliente"
    },
    "AguardarInteracaoCliente": {
      "Type": "Pass",
      "End": true
    },
    "ProcessarPagamento": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ProcessarPagamentoFunction",
        "Payload.$": "$"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "FalhaNoPagamento"
        }
      ],
      "Next": "EnviarParaCozinha"
    },
    "EnviarParaCozinha": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "NotificarRestauranteFunction",
        "Payload.$": "$"
      },
      "Next": "PedidoEmPreparo"
    },
    "PedidoEmPreparo": {
      "Type": "Pass",
      "Result": "Pedido enviado com sucesso para preparo e entrega!",
      "End": true
    },
    "FalhaNoPagamento": {
      "Type": "Fail",
      "Cause": "Erro na cobranca do pedido.",
      "Error": "PagamentoRecusado"
    }
  }
}
