# Changelog

## [Unreleased]
### Added
- Campo `charges` ao objeto `PaymentPix` no `swagger.yaml` para representar juros, multa, descontos e abatimentos.
- Novo schema `PaymentPixCharges` com as estruturas `interest`, `fine`, `discount` e `rebate`.
- Suporte a descontos com datas fixas via `descontoDataFixa`.
- Campos `dueDate` e `latePaymentLimitDate` adicionados em `PaymentPix`.
- Novas validações documentadas no cabeçalho de `Validações` para os campos de encargos e data limite.

