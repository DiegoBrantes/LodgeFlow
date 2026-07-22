# Exemplos

> ⚠️ **Todos os arquivos desta pasta são FICTÍCIOS.**
>
> Foram criados para ilustrar convenções de contrato de API — formato de payload, tratamento de erro,
> idempotência e estrutura de token. **Nenhum** deles representa dados, endpoints, esquemas ou
> respostas reais do LodgeFlow.
>
> Identificadores são valores de teste. Domínios usam `.invalid` ou `.example`, reservados por norma e
> que nunca resolvem. Nenhuma credencial real aparece em nenhum arquivo.

| Arquivo | Demonstra |
|---|---|
| [example-api-request.json](example-api-request.json) | Requisição com idempotência e id gerado no cliente |
| [example-api-response.json](example-api-response.json) | Resposta de sucesso e resposta de erro |
| [example-webhook.json](example-webhook.json) | Evento de webhook com assinatura e id de evento |
| [example-jwt-payload.json](example-jwt-payload.json) | Estrutura de claims de um token |

Consulte [`../api/openapi-example.yaml`](../api/openapi-example.yaml) para o contrato completo — também
fictício.
