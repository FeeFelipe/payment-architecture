# Payment Architecture

# Classificação dos Domínios

| Categoria              | Domínio                        | Descrição                                                                                              |
|------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| **Core Domain**        | **Pagamentos**                 | Responsável por criar, autorizar, liquidar e estornar pagamentos entre contas de pagamento.            |
| **Core Domain**        | **Ledger**                     | Registra lançamentos imutáveis, garantindo integridade contábil e rastreabilidade.    |
| **Supporting Domain**  | **Contas de Pagamento**        | Gerencia o ciclo de vida das contas, vínculo com usuários e disponibilidade de saldo para transações.  |
| **Supporting Domain**  | **Gestão de Usuários**         | Administra o cadastro, perfis, autenticação e status de clientes e lojistas.                           |
| **Supporting Domain**  | **Saldo e Extratos**           | Consolida movimentações derivadas do Ledger e apresenta o histórico e saldo das contas aos usuários.   |
| **Generic Domain**     | **Integrações com Provedores** | Orquestra a comunicação com bancos, PIX e gateways externos, traduzindo comandos e recebendo callbacks.|
| **Generic Domain**     | **Notificações**               | Gera comunicações automáticas (e-mail, push, SMS) baseadas em eventos do sistema.                      |


# Mapa de Contexto

<img width="2899" height="1722" alt="bonded_context" src="https://github.com/user-attachments/assets/b4845512-500a-4d70-b745-c941e49e3c10" />

