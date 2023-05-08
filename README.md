# Rascunho de proposta de IC para 2023/2024

## Tema: Análise da adoção da linguagem Rust no kernel Linux

### Objetivos

- Avaliar a hipótese de que a linguagem Rust traz vantagens de segurança de memória/concorrência para o kernel Linux, oferecendo performance similar ao C
- Contribuir, usando a linguagem Rust, para o código do kernel Linux

### Ideia geral dos trabalhos da IC
- Realizar um estudo extensivo da arquitetura do kernel Linux
    - Particularmente dos modelos de memória, concorrência e segurança
- Fazer um levantamento e análise do histórico de vulnerabilidades encontradas no kernel; identificar as causas mais comuns.
    - Spoiler: [a esmagadora maioria](https://www.cvedetails.com/vulnerability-list/vendor_id-33/Linux.html) são relacionados a corrupção de memória (buffer overflow, use-after-free, null pointer dereference, etc) e/ou condições de corrida em código com concorrência
- Investigar a fundo o processo de adoção do Rust no kernel
    - Como funciona o processo de build do kernel com código em Rust?
    - Dependências do LLVM/clang? Qual o estado atual do backend de GCC para Rust?
    - Quais extensões do compilador são necessárias para compilar o código em Rust do kernel vs. o código em C? (e.g. extensões non-standard, flags de compilador, ABI, etc.)
    - Quais funcionalidades (especialmente instáveis) do Rust estão sendo usadas?
    - Qual o estado atual das abstrações em Rust já presentes no kernel? Que melhorias elas trazem?
- Estudar os casos de uso já existentes
    - [Reescrita do driver de NVMe](https://rust-for-linux.com/nvme-driver)
    - Implementação em Rust do [Binder](https://lore.kernel.org/lkml/20210414184604.23473-14-ojeda@kernel.org/) (mecanismo de IPC do Android)
    - [Kernel drivers](https://github.com/AsahiLinux/linux/tree/asahi/drivers/gpu/drm/asahi) para a GPU dos chips M1 da Apple
    - Implementação de referência [em Rust](https://lore.kernel.org/lkml/20230503090708.2524310-9-nmi@metaspace.dk/T/) do [`null_blk`](https://www.kernel.org/doc/Documentation/block/null_blk.txt), um *block device* usado para testes/benchmarks em várias implementações da *block layer* do kernel.
- Finalmente, realizar contribuições relevantes no ecossistema de Rust no kernel Linux
    - Idealmente, conseguir mandar patches que sejam aceitos no upstream
    - Entrar em contato com a comunidade e os maintainers do subsistema de Rust (Miguel Ojeda, Wedson Almeida Filho) por meio das mailing lists/IRC/Zulip
    - Foco principal dos esforços hoje em dia: device drivers
        - Relativamente isolados, facilmente testáveis, podem ser compilados como módulos do kernel
        - Principais subsistemas: [block layer](https://lore.kernel.org/lkml/20230503090708.2524310-9-nmi@metaspace.dk/T/), [graphics](https://lore.kernel.org/asahi/20230307-rust-drm-v1-0-917ff5bc80a8@asahilina.net/t/) (drm - Direct Rendering Manager)

## Contexto

### Rust

O Rust é uma linguagem de programação orientada a sistemas que busca oferecer duas coisas:

- Garantias de segurança de memória/concorrência em tempo de compilação
- Abstrações de alto nível como *pattern matching*, ferramentas de programação funcional (iteradores, closures, etc.), um *type system* poderoso com *generics* e [sum types](https://en.wikipedia.org/wiki/Tagged_union), etc.; tudo a custo zero

#### Referências seguras

O ponto mais forte do Rust em termos de segurança de memória vem da forma como referências a objetos na memória são tratadas na linguagem.

Considere o seguinte código em C++:

```cpp
#include <cstdio>
#include <vector>

int main() {
    std::vector<int> my_vec = {1, 2, 3, 4};
    const int* my_ptr = &my_vec[0];             // ponteiro pro primeiro elemento de my_vec
    my_vec.push_back(5);                        // insere o número 5 no vetor
    printf("%d\n", *my_ptr);                    // comportamento indefinido!!
}
```

O código à primeira vista parece inocente e correto, mas gera UB (*undefined behavior*); isso acontece porque a classe `std::vector` é um vetor dinâmico que pode ser realocado (para aumentar de tamanho) a qualquer momento. Normalmente o espaço alocado para um `std::vector` é arrendado pra cima, na potência de 2 mais próxima. Quanto o tamanho do `my_vec` passa de 4 pra 5, o vetor é realocado (dobra de tamanho, de 4 pra 8), de modo que os dados são todos copiados para um vetor maior e o antigo é jogado fora.

Desse modo, após inserir o elemento 5, a referência ```*my_ptr = &my_vec[0];``` se torna invalidada porque o vetor é desalocado! A raíz desse comportamento é o fato de que existem duas referências mutáveis para o buffer com os dados do vetor: uma dentro do objeto `std::vec my_vec`, e outra no ponteiro `const int* my_ptr`. A referência `my_ptr` não tem como saber que `my_vec` invalidou o buffer.

Esse tipo de erro é muito comum e difícil de prevenir/identificar (geralmente se identifica usando um *memory sanitizer*, que é difícil se utilizar, por exemplo, em um kernel). A linguagem Rust, contudo, oferece uma solução. Abaixo está a "transcrição" em Rust do código em C++ acima:

```rs
fn main() {
    let mut my_vec = vec![1, 2, 3, 4];
    let my_ptr = &my_vec[0];
    my_vec.push(5);
    println!("{}", &my_ptr);
}
```

Ao compilar o código acima, o compilador imediatamente reclama:

```
error[E0502]: cannot borrow `my_vec` as mutable because it is also borrowed as immutable
 --> test.rs:4:5
  |
3 |     let my_ptr = &my_vec[0];
  |                   ------ immutable borrow occurs here
4 |     my_vec.push(5);
  |     ^^^^^^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", &my_ptr);
  |                    ------- immutable borrow later used here

error: aborting due to previous error

For more information about this error, try `rustc --explain E0502`.
```

Como é possível perceber, o compilador imediatamente identifica o problema e se recusa a permitir que existam simultaneamente uma referência mutável e uma referência imutável apontando para o mesmo objeto. Essa e outras regras que a linguagem impõe evitam uma classe enorme de erros de memória que o compilador é capaz de rapidamente identificar e apontar uma solução.

Outro problema comum pode ser ilustrado pelo seguinte fragmento de código em C:

```c
char* longest(char *a, char *b) {
    if (strlen(a) > strlen(b))
        return a;
    else
        return b;
}

void set_name(char *name, char *input) {
    // for some reason we don't want names shorter than "John"
    char min_name[16];
    strcpy(min_name, "John");
    name = longest(min_name, input); // if we return min_name, it's a dangling pointer
}

int main(int argc, char **argv) {
    char *name;
    set_name(name, "Bob");
    printf("name: %s\n", name); // undefined behavior
    return 0;
}
```

No código acima há undefined behavior porque corremos o risco de retornar uma referência para a string `min_name`, que é desalocada ao final da função `set_name`. Isso acontece porque esquecemos de considerar o "tempo de vida" das strings envolvidas. A solução do Rust para esse problema é o conceito de *lifetimes*:

```rs
// recebe duas referências para strings, retorna uma referência para a mais longa
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() {
        return a;
    } else {
        return b;
    }
}
```

Novamente, o compilador reclama:

```
error[E0106]: missing lifetime specifier
 --> test.rs:1:33
  |
1 | fn longest(a: &str, b: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
  |           ++++     ++          ++          ++

error: aborting due to previous error

For more information about this error, try `rustc --explain E0106`.
```

Ou seja, o compilador exige que nós especifiquemos um lifetime para a variável de saída **em função dos lifetimes das entradas**, justamente para evitar o tipo de bugs de C do código anterior. Isso acontece porque o compilador precisa provar matematicamente a validade de todas as referências no código.

### Vantagens da adoção da linguagem no kernel Linux

Como é possível perceber dos exemplos acima, o design da linguagem Rust é pensado de modo a evitar uma série de erros de memória comuns em C/C++. Além disso, o compilador é excepcionalmente bom em explicar exatamente o erro que foi cometido e sugerir formas de consertá-lo. Em termos de funcionalidades, há uma série de vantagens que a linguagem traz:

- Suporte a estruturas de dados comuns da linguagem C (structs, enums, etc) juntamente com funcionalidades de OOP (`Traits`, similares a interfaces em Java)
    - Estimula *composition over inheritance* e outras boas práticas de OOP
- Pattern matching poderoso (bem mais do que um `switch` comum)
- Suporte a tipos genéricos (`Vector<T>`, etc)
- Suporte a iteradores e closures em tempo de compilação
- Sistema de tipos poderoso e expressivo
- *Error handling* poderoso (tornado possível pelo sistema de tipos)


### Referências

- [\[RFC\] Rust support in the Linux kernel](https://lwn.net/Articles/852659/)
- [Writing Linux Kernel Modules in Safe Rust - Geoffrey Thomas, Linux Security Summit](https://lssna19.sched.com/event/RHaT/writing-linux-kernel-modules-in-safe-rust-geoffrey-thomas-two-sigma-investments-alex-gaynor-alloy)
- [Linux Kernel Module Development in Rust](https://ieeexplore.ieee.org/document/9888822)
- [Rust for Linux - Contributing](https://rust-for-linux.com/contributing)
- [Compiling the Linux Kernel with Rust support](https://www.kernel.org/doc/html/next/rust/quick-start.html)
- [Linux (PCI) NVMe driver in Rust - Andreas Hindborg, Linux Plumbers Conference](https://lpc.events/event/16/contributions/1180/attachments/1017/1961/deck.pdf)
- [Towards Linux Kernel Memory Safety](https://arxiv.org/pdf/1710.06175.pdf)
- [Static verification for memory safety of Linux kernel drivers](https://ispranproceedings.elpub.ru/jour/article/view/1125/849)
- [The Case for Writing a Kernel in Rust](https://patpannuto.com/pubs/levy17rustkernel.pdf)
