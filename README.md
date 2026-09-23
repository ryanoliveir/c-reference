# C — do básico aos structs

Referência de consulta rápida. Cada seção tem a explicação do _porquê_ antes do _como_, e exemplos curtos que podem ser copiados direto. Os exemplos usam dois assuntos recorrentes — rede (IP, porta, SSID, cabeçalho) e arquivos (caminhos, tamanhos, permissões) — apenas como matéria-prima. Nada aqui exige conhecimento prévio de nenhum dos dois.

## Índice

1. [Compilação](#1-compilação)
2. [Tipos, tamanhos e memória](#2-tipos-tamanhos-e-memória)
3. [Operadores e controle de fluxo](#3-operadores-e-controle-de-fluxo)
4. [Funções, escopo e armazenamento](#4-funções-escopo-e-armazenamento)
5. [Ponteiros](#5-ponteiros)
6. [Arrays e strings](#6-arrays-e-strings)
7. [Memória dinâmica](#7-memória-dinâmica)
8. [Structs](#8-structs)
9. [Organizando um projeto](#9-organizando-um-projeto)
10. [Diagnóstico e erros comuns](#10-diagnóstico-e-erros-comuns)

---

## 1. Compilação

`gcc main.c -o main` executa quatro programas diferentes em sequência. Saber qual deles falhou é o que separa ler a mensagem de erro de chutar.

```
main.c --[cpp]--> main.i --[cc1]--> main.s --[as]--> main.o --[ld]--> main
                                                   libc.a / libm.a --^
```

**Pré-processador (cpp).** Manipula texto, só isso. Resolve `#include` colando o arquivo inteiro no lugar, expande `#define`, remove comentários e resolve `#if`/`#ifdef`. Não conhece tipos nem sintaxe de C. Por isso `#define DOBRO(x) x*2` faz `DOBRO(1+1)` virar `1+1*2` = 3 — o remédio é parentizar tudo: `#define DOBRO(x) ((x)*2)`.

**Compilação (cc1).** Aqui existe a linguagem C de verdade: análise sintática, checagem de tipos, otimização. Produz assembly. Erros de sintaxe, de tipo e avisos vêm daqui.

**Assembler (as).** Traduz assembly em código de máquina e gera o objeto `.o`. Quase nunca falha se a etapa anterior passou.

**Linker (ld).** Junta os `.o` e as bibliotecas, resolve cada símbolo (nome de função ou variável global) ao seu endereço final. Não sabe nada sobre tipos — só casa nomes. É por isso que erro de linker é uma categoria à parte: o código compilou, mas alguém prometeu uma função e ninguém a escreveu.

### Parando em cada etapa

| Flag      | Para em         | Saída                    |
| --------- | --------------- | ------------------------ |
| `-E`      | pré-processador | texto expandido (stdout) |
| `-S`      | compilação      | `main.s` (assembly)      |
| `-c`      | assembler       | `main.o` (objeto)        |
| (nenhuma) | linker          | executável               |

Use `gcc -E main.c | less` quando uma macro se comportar de forma estranha: você vê exatamente o que o compilador viu. Use `-S` quando quiser entender o que uma otimização fez.

### Flags que valem a pena memorizar

| Flag            | Para quê                                                                |
| --------------- | ----------------------------------------------------------------------- |
| `-Wall -Wextra` | liga os avisos úteis; não são "all" apesar do nome                      |
| `-Werror`       | trata aviso como erro — bom em CI, incômodo enquanto explora            |
| `-g`            | inclui símbolos de debug, necessário para gdb/valgrind mostrarem linhas |
| `-O0`           | sem otimização (padrão); o que você debuga                              |
| `-O2`           | otimização normal de release                                            |
| `-Os`           | otimiza por tamanho                                                     |
| `-std=c11`      | fixa a versão da linguagem; sem isso o default varia por compilador     |
| `-pedantic`     | reclama de extensões não-padrão do GNU                                  |
| `-I dir`        | onde procurar headers de `#include "..."`                               |
| `-L dir`        | onde procurar bibliotecas                                               |
| `-lm`           | linka `libm` (matemática); a regra é `-lNOME` para `libNOME`            |

Um ponto que pega muita gente: `-lm` precisa vir **depois** dos arquivos que usam a biblioteca, porque o linker processa da esquerda para a direita e só busca o que ainda está pendente. `gcc -lm main.c` falha, `gcc main.c -lm` funciona.

Linha de trabalho razoável para estudar:

```bash
gcc -std=c11 -Wall -Wextra -g -O0 main.c -o main
```

### `#include "..."` vs `#include <...>`

Aspas procuram primeiro no diretório do arquivo atual, depois nos caminhos de sistema. Colchetes angulares pulam o diretório local. Convenção: aspas para os seus headers, angulares para os da biblioteca padrão e de terceiros.

---

## 2. Tipos, tamanhos e memória

Em C, tipo é uma instrução de leitura: quantos bytes ler e como interpretá-los. O padrão define tamanhos **mínimos**, não exatos, e é daí que vem quase todo bug de portabilidade.

| Tipo        | Mínimo garantido | Típico em x86-64       |
| ----------- | ---------------- | ---------------------- |
| `char`      | 1 byte           | 1 byte                 |
| `short`     | 2 bytes          | 2 bytes                |
| `int`       | 2 bytes          | 4 bytes                |
| `long`      | 4 bytes          | 8 bytes (4 no Windows) |
| `long long` | 8 bytes          | 8 bytes                |
| `float`     | —                | 4 bytes                |
| `double`    | —                | 8 bytes                |

Note `long`: 8 bytes no Linux 64-bit, 4 bytes no Windows 64-bit. Código que assume um dos dois quebra no outro.

### Use `stdint.h`

Sempre que o tamanho importar — campo de um cabeçalho de protocolo, formato de arquivo, qualquer coisa que atravessa a rede — use os tipos de largura fixa:

```c
#include <stdint.h>

uint8_t  ttl;        // 0..255   — o campo TTL de um pacote IP tem 1 byte
uint16_t porta;      // 0..65535 — porta TCP/UDP tem 2 bytes
uint32_t ipv4;       // 4 bytes  — um endereço IPv4 cabe exatamente aqui
```

Esses tamanhos não são escolha sua: eles vêm da especificação do protocolo. Escrever `int porta` funciona na sua máquina e vira bug quando o código roda em outra.

### `size_t` e tamanhos de arquivo

`size_t` é o tipo unsigned que o C usa para tamanho de qualquer coisa em memória. É o que `sizeof` retorna e o que `malloc`, `memcpy` e `strlen` recebem. Use ele para contagens e índices de buffer em vez de `int`:

```c
size_t lidos = fread(buf, 1, sizeof buf, arquivo);
for (size_t i = 0; i < lidos; i++) { ... }
```

Tamanho de arquivo é outra história, e um exemplo histórico do que acontece quando o tipo é pequeno demais:

```c
int      tamanho;   // até ~2 GB — quebra em qualquer arquivo maior
uint32_t tamanho;   // até ~4 GB — quebra também
uint64_t tamanho;   // seguro
off_t    tamanho;   // o tipo do sistema para deslocamento em arquivo
```

Um `int` de 32 bits com sinal vai até 2.147.483.647. Um vídeo de 3 GB estoura esse limite e o valor vira negativo — daí a quantidade de programas antigos que simplesmente não abriam arquivos grandes. `off_t` existe justamente para isso: em sistemas de 64 bits ele tem 8 bytes, e em sistemas de 32 bits você o força com `#define _FILE_OFFSET_BITS 64` antes dos includes.

A regra geral: contagem de bytes em memória → `size_t`; posição ou tamanho dentro de um arquivo → `off_t` ou `uint64_t`; nunca `int`.

Existem também `uint_fast8_t` (o mais rápido com pelo menos 8 bits) e `uint_least16_t`. Na prática, `uintN_t` resolve 95% dos casos.

Para imprimir esses tipos com `printf`, use as macros de `inttypes.h`, porque `%d` não serve para todos:

```c
#include <inttypes.h>
printf("porta %" PRIu16 "\n", porta);
```

### Signed, unsigned e overflow

Dois comportamentos diferentes, e a diferença importa:

- **Unsigned** dá a volta de forma definida pelo padrão (aritmética módulo 2ⁿ). `uint8_t x = 255; x++;` resulta em 0, garantido.
- **Signed** estourar é _undefined behavior_. O compilador pode assumir que nunca acontece e otimizar em cima disso. Não é teórico: `if (x + 1 < x)` pode ser eliminado do binário.

A armadilha mais comum é comparar signed com unsigned:

```c
int i = -1;
unsigned u = 1;
if (i < u) { /* NÃO entra aqui */ }
```

O `i` é convertido para unsigned e vira 4294967295. Por isso `-Wall` avisa `comparison of integer expressions of different signedness` — esse aviso merece ser levado a sério.

Esse é exatamente o tipo de descuido que vira vulnerabilidade: uma checagem de tamanho `if (tamanho < limite)` onde `tamanho` é um `int` vindo de um pacote ou do cabeçalho de um arquivo pode passar com valor negativo, e depois virar um número enorme ao ser usado como `size_t` num `malloc` ou num `memcpy`.

O mesmo erro aparece em loop reverso:

```c
for (size_t i = n - 1; i >= 0; i--)  // loop infinito: size_t nunca é < 0
```

### Promoção inteira

Antes de qualquer operação aritmética, tipos menores que `int` são promovidos a `int`. Então:

```c
uint8_t a = 200, b = 100;
uint8_t soma = a + b;              // a+b vira int (300), truncado para 44
uint16_t certo = (uint16_t)a + b;  // 300
```

Em cálculo com deslocamento isso morde: `uint8_t x = 1; x << 9` é calculado em `int`, não em 8 bits.

### Ponto flutuante

`float` e `double` são aproximações binárias. `0.1 + 0.2 != 0.3` — nunca compare floats com `==`:

```c
#include <math.h>
double latencia_ms = 12.5;
if (fabs(latencia_ms - limite) < 1e-9) { /* iguais o bastante */ }
```

Em sistemas sem unidade de ponto flutuante, essa aritmética é emulada em software e custa caro. A saída usual é trabalhar com inteiros em escala fixa (microssegundos em vez de milissegundos fracionários).

### `const`

`const` é uma promessa ao compilador, não uma proteção de memória. Serve para (a) documentar intenção, (b) deixar o compilador reclamar quando você quebrar a promessa, (c) permitir que constantes vão para memória somente-leitura.

```c
const uint16_t PORTA_HTTPS = 443;
void registrar(const char *mensagem);  // a função promete não alterar a mensagem
```

Prefira `const` a `#define` para constantes com valor: `const` tem tipo e escopo, macro é substituição cega de texto.

### Onde cada coisa mora

| Região    | O que guarda                     | Tempo de vida          |
| --------- | -------------------------------- | ---------------------- |
| `.text`   | o código compilado               | todo o programa        |
| `.rodata` | literais de string, `const`      | todo o programa        |
| `.data`   | globais e `static` inicializados | todo o programa        |
| `.bss`    | globais e `static` zerados       | todo o programa        |
| stack     | variáveis locais, argumentos     | até a função retornar  |
| heap      | `malloc`                         | até você chamar `free` |

A stack é pequena e fixa — alguns MB por thread no desktop, bem menos em ambientes restritos. Declarar `uint8_t buffer[1048576]` como local é estouro de stack quase certo; como `static` ou global, vai para `.bss` e funciona.

---

## 3. Operadores e controle de fluxo

C não tem tipo booleano nativo antes de C99: zero é falso, qualquer outro valor é verdadeiro. Isso explica boa parte dos idiomas da linguagem e das armadilhas.

### Bitwise

Trabalhar bit a bit é o motivo de C continuar sendo a linguagem de quem mexe com protocolo: máscara de subrede, flags de cabeçalho e permissões são todos campos de bits.

| Operador | Faz                | Uso típico                         |
| -------- | ------------------ | ---------------------------------- |
| `&`      | E bit a bit        | isolar bits (máscara)              |
| `\|`     | OU bit a bit       | ligar bits                         |
| `^`      | OU exclusivo       | inverter bits, trocar valores      |
| `~`      | complemento        | inverter todos os bits             |
| `<<`     | desloca à esquerda | multiplicar por 2ⁿ, montar máscara |
| `>>`     | desloca à direita  | dividir por 2ⁿ                     |

Os quatro idiomas que você vai usar sempre:

```c
flags |=  (1u << 1);    // liga o bit 1
flags &= ~(1u << 1);    // desliga o bit 1
flags ^=  (1u << 1);    // inverte o bit 1
if (flags & (1u << 1))  // testa o bit 1
```

Use `1u` e não `1`: em `1 << 31` o literal é `int` com sinal e o resultado é undefined behavior. Deslocar por um valor maior ou igual à largura do tipo também é UB — `x << 32` num `uint32_t` não é "zero", é indefinido.

Um endereço IPv4 cabe num `uint32_t`. Extrair os quatro octetos é deslocamento mais máscara:

```c
uint32_t ip = 0xC0A80101;   // 192.168.1.1

uint8_t o1 = (ip >> 24) & 0xFF;   // 192
uint8_t o2 = (ip >> 16) & 0xFF;   // 168
uint8_t o3 = (ip >>  8) & 0xFF;   // 1
uint8_t o4 = (ip      ) & 0xFF;   // 1
```

E o `&` é literalmente como se calcula a rede de um endereço:

```c
uint32_t mascara = 0xFFFFFF00;        // /24, ou 255.255.255.0
uint32_t rede    = ip & mascara;      // 192.168.1.0
uint32_t host    = ip & ~mascara;     // 0.0.0.1

// dois endereços estão na mesma rede?
if ((ip_a & mascara) == (ip_b & mascara)) { ... }
```

Montar a máscara a partir do prefixo CIDR também é deslocamento:

```c
uint32_t mascara_de(int prefixo) {          // prefixo entre 1 e 32
    return 0xFFFFFFFFu << (32 - prefixo);
}
```

O caso `prefixo == 0` precisa ser tratado à parte: deslocar 32 bits num tipo de 32 bits é UB, não zero.

O outro lugar onde máscara aparece o tempo todo é permissão de arquivo. Os `755` e `644` do `chmod` são octal justamente porque cada dígito é um grupo de 3 bits — dono, grupo, outros:

```c
// 0644 = rw- r-- r--
//        110 100 100

if (modo & 0200) { /* o dono pode escrever */ }
if (modo & 0004) { /* qualquer um pode ler  */ }

modo |=  0200;   // dá permissão de escrita ao dono
modo &= ~0077;   // remove todas as permissões de grupo e outros
```

O prefixo `0` em C significa octal, não decimal: `0644` é 420 em decimal, e `644` seria outro número inteiro. Confundir os dois é um erro silencioso, porque as duas formas compilam.

### Precedência: a tabela curta que resolve

De cima para baixo, do que liga mais forte para o mais fraco:

1. `()` `[]` `->` `.`
2. `!` `~` `++` `--` unário `-` `*` (deref) `&` (endereço) `(cast)` `sizeof`
3. `*` `/` `%`
4. `+` `-`
5. `<<` `>>`
6. `<` `<=` `>` `>=`
7. `==` `!=`
8. `&` depois `^` depois `|`
9. `&&` depois `||`
10. `?:`
11. `=` `+=` `-=` ...

Duas consequências práticas que causam bugs reais:

- Bitwise `&` liga **mais fraco** que `==`. Então `if (flags & SYN == 0)` é lido como `if (flags & (SYN == 0))`. Sempre parentize: `if ((flags & SYN) == 0)`.
- `<<` liga mais fraco que `+`. `1 << n + 1` é `1 << (n+1)`.

Regra de bolso: na dúvida, parênteses. Ninguém perde ponto por clareza.

### `&&` `||` e curto-circuito

Avaliação para assim que o resultado é conhecido, e isso é garantido pelo padrão. É o que permite o idioma de checar antes de usar:

```c
if (p != NULL && p->tamanho > 0) { ... }     // seguro
if (i < n && pacote[i] == 0xFF) { ... }      // seguro
```

Inverter a ordem desses testes causa crash.

### `if`, `switch`, loops

```c
switch (estado) {
    case FECHADO:
        iniciar_conexao();
        break;          // sem break, cai no próximo case
    case SYN_ENVIADO:
    case SYN_RECEBIDO:  // dois rótulos, mesmo corpo — fallthrough intencional
        aguardar_handshake();
        break;
    default:
        registrar_erro();
        break;
}
```

`switch` só funciona com inteiros e enums — não com strings nem floats. O `break` esquecido é o bug clássico; `-Wimplicit-fallthrough` avisa.

`while` testa antes, `do/while` testa depois (roda pelo menos uma vez), `for` é açúcar para um `while` com inicialização e incremento no cabeçalho.

### Armadilhas clássicas

**Ponto-e-vírgula solto.** `if (x > 0);` tem corpo vazio; o bloco seguinte roda sempre.

**`=` no lugar de `==`.** `if (autenticado = 1)` atribui e é sempre verdadeiro. Alguns escrevem `if (1 == autenticado)` para que o erro vire erro de compilação; `-Wall` já avisa e é o caminho mais simples.

**Bloco sem chaves.** `if (cond) a(); b();` — `b()` roda sempre. Use chaves sempre, mesmo em uma linha.

**Efeito colateral duplicado.** `vetor[i++] = i;` é undefined behavior: o padrão não define a ordem. Não tente ser esperto com `++` dentro de expressões maiores.

**Divisão inteira.** `1/2` é 0, não 0.5. Para float, force um operando: `1.0/2`.

**Módulo com negativo.** `-7 % 3` é `-1` em C (o sinal segue o dividendo), não `2`.

---

## 4. Funções, escopo e armazenamento

C passa **tudo por valor**. Não existe passagem por referência na linguagem; o que existe é passar o endereço por valor, que dá o mesmo efeito. Entender isso resolve metade das dúvidas sobre ponteiros antes mesmo de chegar neles.

```c
void nao_funciona(int x) { x = 42; }      // altera a cópia
void funciona(int *x)    { *x = 42; }     // altera o original

int a = 0;
nao_funciona(a);   // a continua 0
funciona(&a);      // a vira 42
```

### Declaração vs definição

**Declaração** (protótipo) diz ao compilador que a função existe, qual o tipo de retorno e dos parâmetros. **Definição** é o corpo. O compilador só precisa da declaração para gerar a chamada; o linker é quem exige que exista uma definição.

```c
int conectar(const char *host, uint16_t porta);        // declaração — vai no .h

int conectar(const char *host, uint16_t porta) {       // definição — vai no .c
    ...
}
```

Escreva `void f(void)` e não `void f()` para funções sem parâmetros. Em C, parênteses vazios significam "parâmetros não especificados", não "nenhum parâmetro" — e o compilador não checa nada na chamada.

### Classes de armazenamento

| Palavra                   | Efeito                                                |
| ------------------------- | ----------------------------------------------------- |
| (nenhuma), local          | na stack, morre no fim do bloco                       |
| `static` em local         | vive o programa inteiro, mas só é visível ali dentro  |
| `static` em global/função | símbolo privado ao arquivo `.c` — não entra no linker |
| `extern`                  | "isto existe em outro arquivo", declara sem definir   |
| `register`                | dica obsoleta, o compilador ignora                    |

`static` tem dois significados diferentes dependendo de onde aparece, e isso confunde muita gente:

```c
void registrar_tentativa(void) {
    static int tentativas = 0;   // inicializado uma vez, persiste entre chamadas
    tentativas++;
}

static int conexoes_abertas;             // global privada deste arquivo
static void fechar_socket(int fd) { }    // função privada deste arquivo
```

Em arquivo, `static` é a ferramenta de encapsulamento de C. Como não há namespaces, marcar como `static` tudo que não faz parte da interface pública evita colisão de nomes no link e documenta o que é interno.

### Variável local não é zerada

```c
int x;              // local: lixo, valor indefinido
static int y;       // static/global: garantidamente 0
```

Ler uma local não inicializada é undefined behavior. `-Wall` pega os casos óbvios, mas não todos. Em código que lida com dados externos isso é pior que um bug comum: um buffer local não inicializado e enviado pela rede vaza o que estava na stack.

### Retornar ponteiro para local é erro

```c
char *quebrado(uint32_t ip) {
    char buf[16];
    snprintf(buf, sizeof buf, "%u.%u.%u.%u", ...);
    return buf;      // buf morreu no return — ponteiro pendurado
}
```

O padrão é o chamador fornecer o buffer:

```c
void ip_para_texto(uint32_t ip, char *destino, size_t tamanho) {
    snprintf(destino, tamanho, "%u.%u.%u.%u",
             (ip >> 24) & 0xFF, (ip >> 16) & 0xFF,
             (ip >>  8) & 0xFF,  ip        & 0xFF);
}
```

Esse idioma — quem chama aloca, quem é chamado preenche — aparece em toda a biblioteca padrão e em toda API de sockets, e é o jeito de escrever funções seguras sem depender do heap.

### `inline` e macros

`static inline` num header é a forma moderna de escrever função pequena sem custo de chamada, e é preferível a macro: tem tipos, escopo e avalia os argumentos uma vez só.

```c
static inline int porta_valida(long p) {
    return p > 0 && p <= 65535;
}
```

---

## 5. Ponteiros

Um ponteiro é uma variável comum cujo valor é um endereço. O tipo do ponteiro não muda o que ele guarda — muda quantos bytes ler ao desreferenciar e de quanto andar na aritmética.

```c
uint16_t porta = 443;
uint16_t *p = &porta;   // p guarda o endereço de porta

printf("%u\n", porta);      // 443 — o valor
printf("%p\n", (void*)p);   // 0x7ffd... — o endereço
printf("%u\n", *p);         // 443 — o valor no endereço

*p = 8080;      // porta agora é 8080
```

Dois operadores, sempre inversos: `&` pega o endereço, `*` segue o endereço.

### Como ler uma declaração

Escreva o asterisco colado no nome, não no tipo. O motivo fica claro aqui:

```c
int* a, b;    // a é ponteiro, b é int comum — quase nunca a intenção
int *a, *b;   // os dois são ponteiros
```

O `*` pertence ao declarador, não ao tipo. `int *a` lê-se "`*a` é um int".

### Aritmética de ponteiros

Somar 1 a um ponteiro avança **um elemento**, não um byte. O compilador multiplica pelo `sizeof` do tipo apontado.

```c
uint16_t portas[5] = {22, 80, 443, 3306, 8080};
uint16_t *p = portas;

p + 1     // endereço de portas[1], ou seja, +2 bytes
*(p + 2)  // 443
p[2]      // idêntico — açúcar sintático para *(p + 2)
```

A equivalência `a[i] == *(a + i)` é literal no padrão. Como a soma é comutativa, `2[portas]` compila e vale 443. Curiosidade, não estilo.

Subtrair dois ponteiros dentro do mesmo array dá a distância em elementos, do tipo `ptrdiff_t`. Somar dois ponteiros não faz sentido e não compila.

### Array não é ponteiro

A confusão mais persistente em C. São coisas diferentes que se comportam igual em um contexto.

```c
uint8_t pacote[64];
uint8_t *p = pacote;

sizeof(pacote)   // 64 — o array inteiro
sizeof(p)        // 8  — só o ponteiro
```

O que acontece é o **decay**: em quase toda expressão, o nome de um array vira um ponteiro para seu primeiro elemento. As exceções são `sizeof`, `&` e literais de string em inicialização.

A consequência prática dói: ao passar um array para uma função, o tamanho se perde.

```c
void f(uint8_t pacote[64]) {   // mentira: o 64 é ignorado, isto é uint8_t *
    sizeof(pacote);            // 8, não 64
}

void certo(const uint8_t *pacote, size_t n) { ... }   // sempre passe o tamanho
```

Por isso toda API de C que recebe buffer recebe também o comprimento. Não há alternativa dentro da linguagem — e toda função que processa dados de rede precisa desse parâmetro para não ler além do que chegou.

### `const` com ponteiros

Leia da direita para a esquerda a partir do nome:

```c
const char *p;         // ponteiro para char const — não pode mudar *p
char * const p;        // ponteiro const para char — não pode mudar p
const char * const p;  // nem um nem outro
```

`const char *` é o que você quer em parâmetro de função que só lê. É uma promessa verificada pelo compilador e documenta a interface.

### `void *`

Ponteiro genérico: guarda qualquer endereço mas não pode ser desreferenciado sem cast, porque não há tamanho associado. É o tipo de retorno de `malloc` e o mecanismo de generidade em C.

```c
void *bruto = malloc(1500);
uint8_t *quadro = bruto;    // em C a conversão é implícita, não precisa de cast
```

Em C, fazer cast do retorno de `malloc` é desnecessário e alguns consideram prejudicial. Em C++ é obrigatório — daí vem a divergência de estilo.

### `NULL` e ponteiros pendurados

`NULL` é o endereço que garantidamente não aponta para nada. Desreferenciar dá segfault.

```c
uint8_t *p = NULL;
if (p != NULL) { *p = 1; }   // checar antes é o hábito
```

Depois de `free`, atribua `NULL` ao ponteiro. Não conserta nada por si, mas transforma um bug silencioso (use-after-free) em um crash previsível.

### Ponteiro para ponteiro

Necessário quando a função precisa alterar o próprio ponteiro do chamador:

```c
int alocar_buffer(uint8_t **destino, size_t n) {
    *destino = malloc(n);
    return *destino != NULL;
}

uint8_t *buf = NULL;
alocar_buffer(&buf, 1500);   // buf agora aponta para a memória alocada
```

Mesma lógica de sempre: para alterar um `T` do chamador, receba um `T*`. Se o `T` já é um ponteiro, você recebe `T**`.

### Ponteiro de função

A forma de fazer callback, tabela de despacho e polimorfismo em C.

```c
int somar(int a, int b) { return a + b; }

int (*op)(int, int) = somar;   // declaração: nome entre parênteses
op(2, 3);                      // 5
```

Os parênteses ao redor de `*op` são obrigatórios: sem eles, `int *op(int,int)` é uma função que retorna `int*`.

Um `typedef` deixa legível, e é assim que aparece em código real:

```c
typedef void (*handler_t)(const uint8_t *pacote, size_t tamanho);

void registrar_handler(uint8_t protocolo, handler_t h);
```

Tabela de despacho substituindo um `switch` grande — o jeito usual de escolher o tratador pelo tipo de pacote:

```c
static void (*tratadores[])(const uint8_t *, size_t) = {
    tratar_icmp, tratar_tcp, tratar_udp
};
tratadores[indice](pacote, tamanho);
```

O índice precisa ser validado antes: um valor vindo de fora indexando um array de ponteiros de função é uma das piores falhas possíveis.

---

## 6. Arrays e strings

Não existe tipo string em C. Existe array de `char` terminado por um byte zero (`'\0'`), e toda a biblioteca padrão assume essa convenção. Esquecer o terminador é a origem de uma quantidade impressionante de vulnerabilidades.

```c
char ip[] = "10.0.0.1";   // 9 bytes: 8 caracteres + '\0'
```

`strlen("10.0.0.1")` é 8, mas ocupa 9 bytes. Todo buffer de string precisa de espaço para o terminador. É por isso que o tamanho reservado para um IPv4 em texto é 16 e não 15: `"255.255.255.255"` tem 15 caracteres mais o zero final.

```c
char endereco[16];   // maior IPv4 possível em texto + terminador
```

### Inicialização de arrays

```c
uint8_t a[5] = {1, 2, 3};      // resto vira 0: {1,2,3,0,0}
uint8_t b[5] = {0};            // tudo zero
uint8_t c[]  = {1, 2, 3};      // tamanho deduzido: 3
uint8_t d[5] = {[2] = 7};      // designated initializer (C99): {0,0,7,0,0}
```

C não verifica limites. `a[10]` compila e lê memória arbitrária. Não há rede de segurança — a disciplina é sua.

### Array 2D

```c
int m[3][4];        // 3 linhas, 4 colunas, contíguo na memória
m[1][2] = 5;
```

O layout é row-major: `m[1][2]` está em `*(&m[0][0] + 1*4 + 2)`. Percorrer na ordem das linhas é bem mais rápido por causa do cache.

Para passar a uma função, todas as dimensões menos a primeira são obrigatórias:

```c
void f(int m[][4], size_t linhas);
```

### Literal de string é somente-leitura

```c
char *p = "localhost";   // aponta para .rodata — alterar é UB, geralmente segfault
char v[] = "localhost";  // cópia na stack — pode alterar

p[0] = 'L';   // crash
v[0] = 'L';   // ok
```

Declare literais como `const char *` para que o compilador pegue esse erro.

### Funções de `string.h`

| Função             | O que faz              | Cuidado                            |
| ------------------ | ---------------------- | ---------------------------------- |
| `strlen(s)`        | comprimento sem o `\0` | percorre até achar o zero; O(n)    |
| `strcpy(d, s)`     | copia                  | **sem limite** — estoura o destino |
| `strncpy(d, s, n)` | copia até n            | pode não terminar em `\0`          |
| `strcat(d, s)`     | concatena              | mesmo problema de `strcpy`         |
| `strcmp(a, b)`     | compara                | retorna 0 se iguais, não 1         |
| `memcpy(d, s, n)`  | copia n bytes          | regiões não podem se sobrepor      |
| `memmove(d, s, n)` | copia n bytes          | seguro com sobreposição            |
| `memset(p, c, n)`  | preenche n bytes       | `c` é convertido a `unsigned char` |

Dois pontos que pegam com frequência:

`strcmp` retorna 0 para iguais. `if (strcmp(a, b))` entra no bloco quando são **diferentes**. Escreva `if (strcmp(a, b) == 0)`.

`strncpy` não é a versão segura de `strcpy`. Se a origem tem exatamente `n` caracteres ou mais, o destino fica sem terminador. O idioma é usar `snprintf`:

```c
char ssid[33];   // SSID tem no máximo 32 bytes, mais o terminador
snprintf(ssid, sizeof ssid, "%s", entrada);   // trunca, sempre termina em '\0'
```

`snprintf` é a ferramenta geral de montar string com segurança: ela trunca em vez de estourar e retorna quantos bytes _seriam_ necessários, o que permite detectar truncamento:

```c
int n = snprintf(ssid, sizeof ssid, "%s", entrada);
if (n < 0 || (size_t)n >= sizeof ssid) {
    /* o nome não coube — trate em vez de seguir com dado cortado */
}
```

### `sizeof` em array vs ponteiro

```c
char buf[64];
snprintf(buf, sizeof buf, ...);   // 64 — correto, buf é array aqui

void f(char *buf) {
    snprintf(buf, sizeof buf, ...);   // 8 — BUG, buf é ponteiro
}
```

Esse é o motivo de a função sempre receber o tamanho como parâmetro separado.

### Percorrer string

```c
for (size_t i = 0; s[i] != '\0'; i++) { ... }

// ou, com ponteiro:
for (const char *p = s; *p; p++) { ... }
```

As duas formas são idiomáticas. A segunda evita reindexar e aparece bastante em código de sistema.

### Caminhos de arquivo

Um caminho é só uma string, e montar um a partir de partes é o caso mais comum de concatenação segura:

```c
#include <limits.h>   // PATH_MAX

char caminho[PATH_MAX];
int n = snprintf(caminho, sizeof caminho, "%s/%s", diretorio, nome);
if (n < 0 || (size_t)n >= sizeof caminho) {
    /* não coube — o caminho ficou truncado, não abra o arquivo */
}
```

Nunca use `strcat` para isso. Um diretório longo mais um nome longo passam do buffer sem aviso nenhum, e o resultado é um caminho cortado apontando para outro lugar.

Achar a extensão ou o último componente é trabalho de `strrchr`, que busca a última ocorrência de um caractere:

```c
#include <string.h>

const char *ext = strrchr(caminho, '.');
if (ext != NULL && strcmp(ext, ".log") == 0) { ... }

const char *arquivo = strrchr(caminho, '/');
arquivo = (arquivo != NULL) ? arquivo + 1 : caminho;   // sem '/', o caminho já é o nome
```

Repare que `strrchr` devolve um ponteiro para dentro da string original, não uma cópia. Isso é barato e é o idioma normal em C, mas significa que o resultado só vive enquanto a string original viver — e que alterar um deles altera o outro.

Dois detalhes que mordem: `strrchr(caminho, '.')` em `.bashrc` devolve a string inteira, porque o ponto é o primeiro caractere; e um caminho sem extensão devolve `NULL`, que precisa ser checado antes de qualquer `strcmp`.

### Cores no terminal

São só strings com sequências de escape ANSI, úteis para destacar saída:

```c
#define VERMELHO "\033[31m"
#define VERDE    "\033[32m"
#define RESET    "\033[0m"

printf(VERDE "porta 443 aberta" RESET "\n");
printf(VERMELHO "porta 22 fechada" RESET "\n");
```

O `\033` é o caractere ESC (27 em decimal). Literais de string adjacentes são concatenados pelo compilador, que é o motivo de o exemplo acima funcionar sem nenhuma chamada de função. Sempre emita o `RESET`, senão a cor vaza para o resto do terminal.

---

## 7. Memória dinâmica

Use o heap quando o tamanho só é conhecido em tempo de execução ou quando o dado precisa sobreviver à função que o criou. Fora isso, prefira stack ou `static`: é mais rápido, não fragmenta e não pode vazar.

```c
#include <stdlib.h>

uint8_t *buf = malloc(tamanho);   // não inicializa: lixo
if (buf == NULL) { /* trate a falha */ }

free(buf);
buf = NULL;
```

### As quatro funções

| Função             | Faz             | Nota                                 |
| ------------------ | --------------- | ------------------------------------ |
| `malloc(n)`        | reserva n bytes | conteúdo indefinido                  |
| `calloc(qtd, tam)` | reserva e zera  | detecta overflow na multiplicação    |
| `realloc(p, n)`    | redimensiona    | pode mover o bloco                   |
| `free(p)`          | devolve         | `free(NULL)` é seguro e não faz nada |

Escreva `sizeof *v` em vez de `sizeof(int)`. Se o tipo de `v` mudar depois, o `sizeof` acompanha sozinho e você não tem um bug silencioso de tamanho.

`calloc` tem uma vantagem que não é só a de zerar: ele detecta overflow na multiplicação. `malloc(qtd * tam)` com valores vindos de fora pode dar a volta e alocar um bloco minúsculo para um `qtd` enorme — o passo seguinte é um estouro de buffer.

### O caso mais comum: ler um arquivo para a memória

É onde o heap é obrigatório, porque o tamanho só se conhece em execução:

```c
char *ler_arquivo(const char *caminho, size_t *tamanho_saida) {
    FILE *f = fopen(caminho, "rb");
    if (f == NULL) return NULL;

    fseek(f, 0, SEEK_END);       // vai até o fim
    long tamanho = ftell(f);     // a posição atual é o tamanho
    rewind(f);                   // volta para o início

    if (tamanho < 0 || tamanho > 100L * 1024 * 1024) {   // limite explícito
        fclose(f);
        return NULL;
    }

    char *buf = malloc((size_t)tamanho + 1);   // +1 para o terminador
    if (buf == NULL) { fclose(f); return NULL; }

    size_t lidos = fread(buf, 1, (size_t)tamanho, f);
    fclose(f);

    buf[lidos] = '\0';           // permite tratar como string
    *tamanho_saida = lidos;
    return buf;                  // quem chamou é dono: precisa dar free
}
```

Quatro decisões deste código valem mais que o código:

- O limite de tamanho não é paranoia. Sem ele, apontar a função para um arquivo de 40 GB é um `malloc` de 40 GB. Todo tamanho que vem de fora precisa de um teto.
- O `+1` e o `buf[lidos] = '\0'` são o que permitem passar o resultado para `strlen` e companhia. Sem isso você tem bytes, não string.
- Guardo `lidos`, não `tamanho`: em leitura de texto no Windows os dois diferem, e confiar no `ftell` para indexar o buffer lê lixo no fim.
- A função retorna memória alocada, então o contrato de quem libera precisa estar escrito no header. Aqui é quem chama.

### `realloc` corretamente

```c
uint8_t *novo = realloc(buf, novo_tamanho);
if (novo == NULL) {
    free(buf);       // buf ainda é válido aqui — realloc falhou
    return -1;
}
buf = novo;
```

O erro comum é `buf = realloc(buf, ...)`. Se falhar, `buf` vira `NULL` e o bloco original vaza — você perdeu a única referência a ele.

### Os quatro erros clássicos

**Vazamento (leak).** Alocou e não liberou. Não trava nada imediatamente; o processo cresce até morrer. Num serviço que fica meses no ar, um leak por conexão atendida é questão de tempo.

**Use-after-free.** Usar o ponteiro depois do `free`. O bloco pode já ter sido reciclado por outra alocação — os dados parecem corretos por um tempo e depois não parecem mais. Atribuir `NULL` após `free` converte isso em crash imediato.

**Double free.** Liberar duas vezes. Corrompe as estruturas internas do alocador; o crash acontece longe da causa. `free(NULL)` é seguro, então zerar o ponteiro também protege aqui.

**Estouro de buffer.** Escrever além do que foi alocado. Sobrescreve os metadados do heap e o programa quebra em outro lugar, muito depois.

O padrão comum dos três últimos: o sintoma aparece longe da causa. É exatamente por isso que valgrind e AddressSanitizer existem (seção 10).

### Regras que evitam a maior parte disso

1. Uma alocação, um dono. Defina de quem é a responsabilidade de liberar, e escreva isso em comentário no header.
2. Para cada `malloc`, escreva o `free` correspondente antes de continuar programando.
3. `free(p); p = NULL;` sempre juntos.
4. Se a função aloca, ela expõe também a função que libera (`criar_x` / `destruir_x`).
5. Nunca calcule um tamanho de alocação a partir de um valor externo sem validar o intervalo primeiro.

### Quando evitar o heap

Alocação e liberação repetidas fragmentam: existe memória livre suficiente no total, mas nenhum bloco contíguo do tamanho pedido, e o `malloc` falha sem que nada tenha vazado. Em processos de vida longa e em ambientes com pouca RAM, isso decide o projeto.

Alternativas usuais:

- **Buffers estáticos com tamanho máximo.** Previsível, verificável em tempo de compilação. Um buffer de 1500 bytes cobre qualquer quadro Ethernet padrão.
- **Pool de objetos.** Aloca um array fixo uma vez, gerencia índices livres você mesmo.
- **Alocar tudo na inicialização e nunca liberar.** Muito comum e perfeitamente legítimo.

---

## 8. Structs

Uma struct agrupa campos de tipos diferentes num único valor com layout previsível. É o mecanismo de abstração de dados de C: sem ela, uma função que trabalha com "uma conexão" precisaria de cinco parâmetros soltos.

```c
struct endereco {
    uint32_t ip;
    uint16_t porta;
};

struct endereco a = {0xC0A80101, 443};
struct endereco b = {.ip = 0xC0A80101, .porta = 443};   // C99, prefira esta forma

a.porta = 8080;
```

A forma com nomes de campo (_designated initializer_) não depende da ordem e sobrevive a uma reordenação futura da struct. Campos omitidos viram zero.

### `typedef`

Sem `typedef` você repete `struct` em toda declaração. Com ele, o tipo ganha um nome só:

```c
typedef struct {
    uint32_t ip;
    uint16_t porta;
    uint8_t  protocolo;
} endereco_t;

endereco_t destino = {.porta = 443};
```

Se a struct precisar se referir a si mesma, ela precisa de tag:

```c
typedef struct no {
    endereco_t valor;
    struct no *proximo;   // a tag é obrigatória aqui
} no_t;
```

Há uma discussão real de estilo aqui. O kernel Linux desencoraja `typedef` de struct porque esconde do leitor que aquilo é um agregado; APIs de biblioteca costumam usar porque fica mais limpo. Escolha uma e mantenha.

### `.` e `->`

```c
endereco_t e;
endereco_t *pe = &e;

e.porta = 443;      // valor direto
pe->porta = 443;    // ponteiro
(*pe).porta = 443;  // idêntico ao anterior, mas ninguém escreve assim
```

`->` é só abreviação de desreferenciar e acessar. Use sempre que tiver um ponteiro.

### Structs aninhadas e arrays

```c
typedef struct {
    endereco_t origem;
    endereco_t destino;
    uint32_t   bytes;
} conexao_t;

conexao_t ativas[64];
ativas[3].destino.porta = 443;
```

O mesmo padrão descrevendo uma entrada de diretório:

```c
typedef struct {
    char     nome[256];
    uint64_t tamanho;
    uint16_t modo;       // as permissões da seção 3
    int      e_diretorio;
} entrada_t;

entrada_t listagem[128];
listagem[0].tamanho = 4096;
```

Repare que `nome` é um array dentro da struct, não um ponteiro: a string vive dentro da própria struct, e copiar a struct copia o nome junto. Com `char *nome`, a cópia compartilharia o mesmo texto — e liberar uma das duas deixaria a outra com ponteiro pendurado.

### Passar struct para função

Atribuição de struct copia todos os campos, e isso vale também na passagem de parâmetro. Structs grandes copiadas por valor custam caro.

```c
void por_valor(conexao_t c);               // copia a struct inteira
void por_ponteiro(conexao_t *c);           // copia 8 bytes, pode alterar
void somente_leitura(const conexao_t *c);  // copia 8 bytes, promete não alterar
```

Regra prática: se a struct for maior que dois ou três ponteiros, passe `const T *`. Retornar struct por valor, por outro lado, é normal e o compilador costuma otimizar.

### Padding e alinhamento

O compilador insere bytes vazios para que cada campo caia num endereço alinhado ao seu tamanho. Isso significa que `sizeof` de uma struct não é a soma dos campos.

```c
struct ruim {       // 12 bytes
    uint8_t  ttl;   // 1 byte + 3 de padding
    uint32_t ip;    // 4
    uint8_t  proto; // 1 + 3 de padding
};

struct bom {        // 8 bytes
    uint32_t ip;
    uint8_t  ttl;
    uint8_t  proto; // + 2 de padding no fim
};
```

Ordenar os campos do maior para o menor costuma eliminar a maior parte do desperdício. Numa tabela com milhares de entradas isso importa de verdade.

Duas consequências que causam bugs:

- **Nunca compare structs com `memcmp`.** Os bytes de padding têm conteúdo indefinido; duas structs com campos idênticos podem diferir. Compare campo a campo.
- **Nunca despeje uma struct direto num socket ou arquivo.** O layout varia entre compiladores, arquiteturas e flags — e o padding vai junto, vazando bytes da memória do processo. Serialize campo a campo, com a ordem de bytes explícita.

`#pragma pack(1)` remove o padding, mas gera acesso desalinhado — lento em x86 e exceção de hardware em algumas arquiteturas ARM. Use só para mapear formato binário externo, ciente do custo.

### Bitfields

Empacotam campos de largura arbitrária. Cabeçalhos de protocolo são cheios deles:

```c
typedef struct {
    uint8_t fin : 1;
    uint8_t syn : 1;
    uint8_t rst : 1;
    uint8_t psh : 1;
    uint8_t ack : 1;
    uint8_t urg : 1;
    uint8_t reservado : 2;
} flags_tcp_t;
```

A ordem dos bits dentro da palavra é definida pela implementação. Funcionam bem para economizar memória dentro do seu programa; para interpretar bytes que chegaram da rede, máscaras com `<<` e `&` são mais portáveis e previsíveis:

```c
#define TCP_FIN 0x01
#define TCP_SYN 0x02
#define TCP_ACK 0x10

if ((flags & TCP_SYN) && !(flags & TCP_ACK)) { /* início de handshake */ }
```

### `union`

Todos os campos compartilham o mesmo espaço; o tamanho é o do maior. Serve para representar "um de vários" ou reinterpretar bytes:

```c
typedef union {
    uint32_t inteiro;
    uint8_t  octeto[4];
} ipv4_t;

ipv4_t end;
end.inteiro = 0xC0A80101;
// end.octeto[0] é 0x01 em máquina little-endian, 0xC0 em big-endian
```

O padrão C permite ler um membro diferente do último escrito (_type punning_), diferente de C++. Mas o resultado depende da ordem de bytes da máquina, e é justamente por isso que protocolos definem a sua própria: o que vai na rede é big-endian, o que está no seu x86 é little-endian, e converter entre os dois (`htons`, `htonl`) não é opcional.

O uso mais comum é a união marcada, o jeito de C fazer tipo-soma:

```c
typedef struct {
    enum { ENDERECO_V4, ENDERECO_V6, ENDERECO_NOME } tipo;
    union {
        uint32_t v4;
        uint8_t  v6[16];
        char    *nome;
    } dado;
} alvo_t;
```

Você é responsável por ler o campo que corresponde ao `tipo`. O compilador não checa isso — ler o campo errado de uma union é como o programa interpreta bytes que não são o que ele pensa.

### `enum`

Constantes inteiras nomeadas, começando em 0 e incrementando:

```c
typedef enum {
    CONN_FECHADA,        // 0
    CONN_ESCUTANDO,      // 1
    CONN_ESTABELECIDA,   // 2
    CONN_ERRO = 10       // 10
} estado_conexao_t;
```

Melhor que `#define` para conjuntos de constantes: o depurador mostra o nome em vez do número, e alguns compiladores avisam quando um `switch` não cobre todos os valores do enum. O tipo subjacente é definido pela implementação, então não conte com um tamanho específico dentro de uma struct que mapeia protocolo.

### Opaque pointer — encapsulamento em C

O idioma que substitui classe privada. No header você declara o tipo sem revelar os campos:

```c
// tabela.h
typedef struct tabela tabela_t;      // tipo incompleto

tabela_t *tabela_criar(size_t capacidade);
void      tabela_destruir(tabela_t *t);
int       tabela_inserir(tabela_t *t, uint32_t ip, uint16_t porta);
int       tabela_contem(const tabela_t *t, uint32_t ip);
```

```c
// tabela.c
struct tabela {                      // definição só aqui
    endereco_t *entradas;
    size_t      usadas, capacidade;
};
```

Quem inclui o header pode declarar `tabela_t *`, mas não pode acessar campos nem saber o `sizeof` — só manipular pelas funções. Isso permite mudar a implementação sem recompilar quem usa, e é a base de praticamente toda biblioteca C bem projetada.

---

## 9. Organizando um projeto

A divisão entre `.h` e `.c` segue uma regra só: o header é a interface, o `.c` é a implementação. Quem inclui o header precisa saber o mínimo para chamar suas funções, e nada além disso.

| Vai no `.h`                                     | Vai no `.c`           |
| ----------------------------------------------- | --------------------- |
| protótipos das funções públicas                 | corpo das funções     |
| `typedef` e structs que o usuário precisa criar | structs opacas        |
| `#define` e `enum` da interface                 | constantes internas   |
| `extern` de globais públicas                    | definição das globais |
| —                                               | tudo que é `static`   |

**Nunca defina variável ou função (com corpo) num header.** Cada `.c` que o incluir gera um símbolo, e o linker reclama de definição múltipla. A exceção é `static inline`.

### Include guard

Todo header precisa de proteção contra inclusão dupla:

```c
#ifndef TABELA_H
#define TABELA_H

/* conteúdo */

#endif /* TABELA_H */
```

`#pragma once` faz o mesmo em uma linha e é suportado por todos os compiladores relevantes, mas não está no padrão. As duas opções são defensáveis.

Com header próprio, prefira **forward declaration** a `#include` sempre que só precisar do ponteiro: acelera a compilação e evita dependências circulares.

### Compilação separada

```bash
gcc -c main.c    -o main.o       # cada .c vira um .o
gcc -c tabela.c  -o tabela.o
gcc main.o tabela.o -o scanner   # o linker junta
```

O ganho é que alterar `main.c` não obriga a recompilar `tabela.c`. Em projeto grande isso é a diferença entre segundos e minutos.

### Um Makefile mínimo

```makefile
CC      = gcc
CFLAGS  = -std=c11 -Wall -Wextra -g -O0
LDFLAGS =

ALVO    = scanner
FONTES  = main.c tabela.c rede.c
OBJETOS = $(FONTES:.c=.o)      # substitui .c por .o na lista

all: $(ALVO)

$(ALVO): $(OBJETOS)
	$(CC) $(OBJETOS) -o $@ $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJETOS) $(ALVO)

.PHONY: all clean
```

O que cada parte faz:

- `alvo: dependências` seguido das receitas — o make só executa a receita se alguma dependência for mais nova que o alvo. É isso que evita recompilar o que não mudou.
- `$@` é o nome do alvo, `$<` é a primeira dependência, `$^` são todas.
- `%.o: %.c` é uma regra padrão: vale para qualquer par de arquivos com esses sufixos.
- **As receitas precisam de TAB, não espaços.** É o erro número um de quem começa com make.
- `.PHONY` declara alvos que não produzem arquivo com aquele nome, para o make não confundir com um arquivo chamado `clean`.

Falta uma coisa importante: o make não sabe que `main.c` depende de `tabela.h`. Editar o header não dispara recompilação. A solução automática é gerar as dependências com o próprio compilador:

```makefile
CFLAGS += -MMD -MP
-include $(OBJETOS:.o=.d)
```

`-MMD` faz o gcc escrever um `.d` com as dependências de header de cada `.o`, e o `-include` as carrega. O hífen na frente evita erro na primeira compilação, quando os `.d` ainda não existem.

---

## 10. Diagnóstico e erros comuns

O segredo de depurar C é aceitar que o sintoma quase nunca está onde está a causa. Corrupção de memória quebra o programa _depois_, em outro lugar. Ferramenta boa reduz essa distância.

### AddressSanitizer — comece por aqui

```bash
gcc -fsanitize=address,undefined -g -O1 main.c -o main
./main
```

O ASan instrumenta o binário e aborta na hora exata do estouro, do use-after-free ou do vazamento, com a pilha da alocação e da liberação. Roda uns 2x mais lento, o que é irrelevante enquanto você estuda. O UBSan pega overflow de signed, shift inválido e desalinhamento.

Não combine com valgrind — escolha um dos dois por execução.

### valgrind

```bash
valgrind --leak-check=full ./programa
```

Não exige recompilar, o que o torna útil em binário de terceiro. Mais lento que o ASan (10 a 50x) e detecta um conjunto ligeiramente diferente de problemas.

### gdb — o mínimo que resolve

Compile com `-g -O0` e rode `gdb ./programa`.

| Comando                 | Faz                                                                      |
| ----------------------- | ------------------------------------------------------------------------ |
| `run` (`r`)             | inicia                                                                   |
| `break main.c:42` (`b`) | breakpoint em arquivo:linha                                              |
| `next` (`n`)            | próxima linha, sem entrar em funções                                     |
| `step` (`s`)            | próxima linha, entrando                                                  |
| `continue` (`c`)        | segue até o próximo breakpoint                                           |
| `print x` (`p`)         | mostra o valor; `p *p`, `p v[3]`, `p s.campo`                            |
| `x/16xb p`              | despeja 16 bytes em hexa a partir de `p` — útil para inspecionar buffers |
| `backtrace` (`bt`)      | pilha de chamadas — o primeiro comando após um crash                     |
| `frame 2` (`f`)         | muda para outro nível da pilha                                           |
| `watch x`               | para quando `x` mudar de valor                                           |
| `info locals`           | todas as locais do frame atual                                           |

Depois de um segfault, `bt` sozinho já resolve a maioria dos casos. `watch` é a arma contra "alguém está sobrescrevendo essa variável e não sei quem".

### Traduzindo mensagens

| Mensagem                                   | O que realmente aconteceu                                                        |
| ------------------------------------------ | -------------------------------------------------------------------------------- |
| `implicit declaration of function 'foo'`   | faltou o `#include` ou o protótipo                                               |
| `undefined reference to 'foo'`             | erro de **linker**: o protótipo existe, o corpo não — faltou um `.c` ou uma `-l` |
| `multiple definition of 'foo'`             | definiu no header, ou compilou o mesmo `.c` duas vezes                           |
| `expected ';' before ...`                  | o erro está na linha **anterior**                                                |
| `dereferencing pointer to incomplete type` | usou campos de uma struct só declarada, não definida                             |
| `assignment discards 'const' qualifier`    | tentou escrever através de um ponteiro `const`                                   |
| `control reaches end of non-void function` | falta `return` em algum caminho                                                  |
| `Segmentation fault`                       | desreferenciou `NULL`, ponteiro inválido ou estourou a stack                     |

A distinção entre erro de compilador e erro de linker vale internalizar: `undefined reference` nunca é problema de sintaxe, é ausência de código ou de biblioteca na linha de comando.

### Hábitos que economizam horas

1. Compile sempre com `-Wall -Wextra` e corrija todo aviso. Em C, aviso ignorado vira bug de produção.
2. Leia o **primeiro** erro, não o último. Um erro de sintaxe gera cascata; os seguintes costumam ser fantasmas.
3. Rode com ASan de vez em quando, mesmo quando está tudo funcionando.
4. Todo dado que vem de fora do programa — arquivo, entrada do usuário, rede — é suspeito até ser validado em tamanho e intervalo. A maioria das falhas de memória em C começa com um valor externo usado sem checagem.
5. `printf` de depuração precisa de `fflush(stdout)` ou `\n`, senão o buffer não esvazia antes do crash e você perde a última linha — justamente a que importa.

### Onde consultar

- `man 3 printf`, `man 3 malloc`, `man 2 socket` — a documentação está instalada na sua máquina. Seção 2 é chamada de sistema, seção 3 é função de biblioteca.
- [cppreference — seção C](https://en.cppreference.com/w/c) — a referência mais precisa e navegável do padrão.
- _The C Programming Language_ (Kernighan & Ritchie), 2ª edição — curto, denso, ainda o melhor ponto de partida. Os exercícios valem mais que o texto.
- _Modern C_, de Jens Gustedt — gratuito e atualizado para C11/C17; bom complemento ao K&R, que é anterior a boa parte do que está neste guia.

## Recursos visuais de C

### Visualizar memória (stack, heap, ponteiros)

- [Python Tutor (C/C++)](https://pythontutor.com) - executa C passo a passo mostrando stack e heap
- [Python Tutor: guia do visualizador C/C++](https://pythontutor.com/articles/c-cpp-visualizer.html)
- [CMemoryViz](https://github.com/YheChen/CMemoryViz) - diagrama de memória com endereços, roda no navegador
- [cpp-tutor](https://github.com/jmanoj0905/cpp-tutor) - estilo Python Tutor, roda local via Docker
- [memory-viewer](https://github.com/arana-rs/memory-viewer) - stack, heap e ponteiros passo a passo (em espanhol)
- [C Stack & Heap Memory Visualizer](https://circuitlabs.net/labs/c-stack-heap-memory-visualizer-learning-tool/)
- [Memviz (VS Code)](https://marketplace.visualstudio.com/items?itemName=jakub-beranek.memviz) - memória do seu próprio programa via GDB

### Cursos e explicações animadas

- [Log2Base2](https://log2base2.com) - cursos animados de C e Advanced Pointers
- [Memory Allocation, de Sam Rose](https://samwho.dev/memory-allocation/) - como malloc/free funcionam por dentro
- [Hello Algo](https://www.hello-algo.com) - algoritmos com animações e código em C

### Do C ao assembly

- [Compiler Explorer](https://godbolt.org) - mostra o assembly gerado pelo seu código C

## Animações e visualização de algoritmos

### Plataformas com animação passo a passo

- [AlgoMaster - Animações](https://algomaster.io/animations) - mais de mil animações de DSA, system design e concorrência
- [Algorithm Visualizer](https://algorithm-visualizer.org) - código e animação lado a lado, destaca a linha em execução
- [VisuAlgo](https://visualgo.net/en) - animações com entrada própria, pseudocódigo e quizzes
- [Data Structure Visualizations (USFCA)](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html) - árvores, heaps e hash interativos
- [Hello Algo](https://www.hello-algo.com) - livro aberto com animações em cada conceito
- [Log2Base2](https://log2base2.com) - cursos totalmente animados de programação e DSA

### Temas específicos

- [Sorting Algorithms (Toptal)](https://www.toptal.com/developers/sorting-algorithms) - comparação animada de ordenações
- [PathFinding.js](https://qiao.github.io/PathFinding.js/visual/) - A\*, BFS, Dijkstra em grade interativa
- [Red Blob Games - A\*](https://www.redblobgames.com/pathfinding/a-star/introduction.html) - busca em grafos explicada visualmente
- [Algorithm Visualizations](https://algorithm-visualizations-benjaminjohnson2204.vercel.app/) - listas ligadas e arrays animados com D3

### Explicadores interativos

- [samwho.dev](https://samwho.dev) - hashing, filas, balanceamento de carga, memória
- [Bartosz Ciechanowski](https://ciechanow.ski) - hardware e conceitos técnicos com animações detalhadas
- [Free System Design](https://freesystemdesign.com/) - monte arquiteturas arrastando componentes e simule tráfego (grátis)
- [ByteByteGo](https://bytebytego.com) - system design com diagramas animados
- [Brilliant](https://brilliant.org) - cursos interativos de programação e CS (pago)
