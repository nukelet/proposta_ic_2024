# Rascunho de proposta de IC para 2024

## Tema: Análise da adoção da linguagem Rust no kernel Linux

### Objetivos

Avaliar a hipótese de que a linguagem Rust:

- Traz vantagens reais do ponto de vista de **segurança de memória** e **concorrência** no código do kernel
- É capaz de oferecer performance/memory footprint similar ao código em linguagem C já existente no kernel

### Ideia geral dos trabalhos da IC
- Realizar um estudo extensivo da arquitetura do kernel Linux
    - Particularmente dos modelos de memória, concorrência e segurança
- Fazer um levantamento e análise do histórico de vulnerabilidades encontradas no kernel; identificar as causas mais comuns.
    - Spoiler: [a esmagadora maioria](https://www.cvedetails.com/vulnerability-list/vendor_id-33/Linux.html) são relacionados a corrupção de memória (buffer overflow, use-after-free, null pointer dereference, etc) e/ou condições de corrida em código com concorrência
- Investigar a fundo o processo de adoção do Rust no kernel
    - Como funciona o processo de build do kernel com código em Rust (dependências do LLVM/clang? estado atual do backend de GCC para Rust? quais *hacks* são necessários para compilar o código em Rust do kernel vs. quais *hacks* necessários para compilar o código em C? quais funcionalidades instáveis do Rust estão sendo usadas?)
    - Qual o estado atual das abstrações em Rust (wrappers) das interfaces já existentes do kernel (files, )
