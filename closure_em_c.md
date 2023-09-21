# Closure
Uma closure é um tipo de objeto que contém um ponteiro ou referência 
de algum tipo para uma função a ser executada 
junto com uma instância dos dados necessários para a função.

Um exemplo em JS da [Mozilla](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) é

```js
function makeAdder(x) {
  return function(y) { // create the adder function and return it along with
    return x + y;      // the captured data needed to generate its return value
  };
}
```

que poderia então ser usado como:

```js
var add5 = makeAdder(5);  // create an adder function which adds 5 to its argument

console.log(add5(2));  // displays a value of 2 + 5 or 7
```

# Alguns dos obstáculos a serem superados com C

A linguagem de programação C é uma linguagem de tipagem estática, 
diferente do JavaScript, nem possui coleta de lixo, 
e alguns outros recursos que facilitam a realização de closures 
em JavaScript ou outras linguagens com suporte intrínseco para closures.

Um grande obstáculo para closures no padrão C 
é a falta de suporte da linguagem para o tipo de construção 
como no exemplo em JS,
em que a closure inclui não apenas a função, 
mas também uma cópia dos dados que são capturados quando a closure é criada, 
uma forma de estado de salvamento 
que pode ser usado quando o encerramento é executado 
junto com quaisquer argumentos adicionais fornecidos 
no momento em que a função de encerramento é invocada.

No entanto, C tem alguns blocos de construção básicos 
que podem fornecer as ferramentas para criar um tipo de fechamento. 
Contudo, algumas das dificuldades são: 
(1) o gerenciamento de memória é dever do programador, nenhuma coleta de lixo, 
(2) funções e dados são separados, nenhuma classe ou mecânica de tipo de classe, 
(3) tipado estaticamente, então não há descoberta de tipos de dados em tempo de execução ou tamanhos de dados e 
(4) recursos de linguagem deficientes para capturar dados de estado no momento em que a closure é criada.

Uma coisa que torna possível um recurso de closure com C 
é o void *ponteiro e o uso unsigned char
como uma espécie de tipo de memória de uso geral 
que é então transformado em outros tipos por meio de conversão.


# Uma implementação com o padrão C e um pouco de alongamento aqui e ali

**NOTA** : 
O exemplo a seguir depende de uma convenção de passagem de argumento baseada em pilha, 
como é usada com a maioria dos compiladores x86 de 32 bits. 
A maioria dos compiladores também permite que uma convenção de chamada 
seja especificada além da passagem de argumentos baseados em pilha, 
como o  modificador do Visual Studio __fastcall()__. 
O padrão para x64 e Visual Studio de 64 bits é usar a  __fastcall()__ 
convenção por padrão para que os argumentos de função sejam passados em registradores e não na pilha. 
Consulte [Visão geral das convensões de chamda x64](https://msdn.microsoft.com/en-us/library/ms235286.aspx) 
no Microsoft MSDN, 
bem como [Como definir argumentos de função no assembly durante o tempo de execução em um aplicativo de 64 bits no Windows?](https://stackoverflow-com.translate.goog/questions/49283616/how-to-set-function-arguments-in-assembly-during-runtime-in-a-64bit-application?_x_tr_sl=en&_x_tr_tl=pt&_x_tr_hl=pt-BR&_x_tr_pto=sc) 
bem como as várias respostas e comentários em 
[Como os argumentos variáveis são implementados no gcc?](https://stackoverflow-com.translate.goog/questions/12371450/how-are-variable-arguments-implemented-in-gcc?_x_tr_sl=en&_x_tr_tl=pt&_x_tr_hl=pt-BR&_x_tr_pto=sc)

Uma coisa que podemos fazer para resolver esse problema 
de fornecer algum tipo de facilidade de closure para C 
é simplificar o problema. 
É melhor fornecer uma solução de 80% que seja útil para a maioria dos aplicativos do que nenhuma solução.

Uma dessas simplificações 
é suportar apenas funções que não retornam um valor, 
ou seja, 
funções declaradas como __void_func_name()__.  
Também vamos desistir da verificação de tipo em tempo de compilação 
da lista de argumentos da função, 
pois essa abordagem cria a lista de argumentos da função em tempo de execução. 
Nenhuma dessas coisas que estamos desistindo são triviais, 
então a questão é se o valor dessa abordagem para closures em C 
supera o que estamos desistindo.

Primeiro de tudo vamos definir nossa área de dados de clsoure. 
A área de dados de closure 
representa a área de memória que usaremos 
para conter as informações necessárias para um closure. 
A quantidade mínima de dados que consigo pensar 
é um ponteiro para a função a ser executada 
e uma cópia dos dados a serem fornecidos à função como argumentos.

Nesse caso, forneceremos 
quaisquer dados de estado capturados 
necessários para a função 
como um argumento para a função.

Também queremos ter algumas proteções básicas 
para que possamos falhar com segurança razoável. 
Infelizmente, os trilhos de segurança são um pouco fracos 
com algumas das soluções que estamos usando para implementar uma forma de closure


# O código fonte

O código-fonte a seguir foi desenvolvido usando o Visual Studio 2017 Community Edition em um arquivo de origem .c C.

A área de dados é uma estrutura que contém alguns dados de gerenciamento, um ponteiro para a função e uma área de dados aberta.

```
typedef struct {
    size_t  nBytes;    // current number of bytes of data
    size_t  nSize;     // maximum size of the data area
    void(*pf)();       // pointer to the function to invoke
    unsigned char args[1];   // beginning of the data area for function arguments
} ClosureStruct;
```

Em seguida, criamos uma função que inicializará uma área de dados de fechamento.

```
ClosureStruct * beginClosure(void(*pf)(), int nSize, void *pArea)
{
    ClosureStruct *p = pArea;

    if (p) {
        p->nBytes = 0;      // number of bytes of the data area in use
        p->nSize = nSize - sizeof(ClosureStruct);   // max size of the data area
        p->pf = pf;         // pointer to the function to invoke
    }

    return p;
}
```

Esta função é projetada para aceitar um ponteiro para uma área de dados que dá flexibilidade de como o usuário da função deseja gerenciar a memória. Eles podem usar alguma memória na pilha ou memória estática ou podem usar memória heap por meio da __malloc()__ função.

```
unsigned char closure_area[512];
ClosureStruct *p = beginClosure (xFunc, 512, closure_area);
```
ou
```
ClosureStruct *p = beginClosure (xFunc, 512, malloc(512));
//  do things with the closure
free (p);  // free the malloced memory.
```
Em seguida, fornecemos uma função que nos permite adicionar dados e argumentos ao nosso encerramento. O propósito desta função é construir os dados de fechamento para que quando a função de fechamento seja invocada, a função de fechamento receba todos os dados necessários para realizar seu trabalho.
```
ClosureStruct * pushDataClosure(ClosureStruct *p, size_t size, ...)
{
    if (p && p->nBytes + size < p->nSize) {
        va_list jj;

        va_start(jj, size);    // get the address of the first argument

        memcpy(p->args + p->nBytes, jj, size);  // copy the specified size to the closure memory area.
        p->nBytes += size;     // keep up with how many total bytes we have copied
        va_end(jj);
    }

    return p;
}
```
E para tornar isso um pouco mais simples de usar, vamos fornecer uma macro de encapsulamento que geralmente é útil, mas tem limitações, pois é a manipulação de texto do processador C.
```
#define PUSHDATA(cs,d) pushDataClosure((cs),sizeof(d),(d))
```
para que pudéssemos usar algo como o seguinte código-fonte:
```
unsigned char closurearea[256];
int  iValue = 34;

ClosureStruct *dd = PUSHDATA(beginClosure(z2func, 256, closurearea), iValue);
dd = PUSHDATA(dd, 68);
execClosure(dd);
```

# Invocando o encerramento: a função execClosure()

A última parte disso é a __execClosure()__ função para executar a função de fechamento com seus dados. O que estamos fazendo nesta função é copiar a lista de argumentos fornecida na estrutura de dados de fechamento para a pilha enquanto invocamos a função.

O que fazemos é converter a área args dos dados de encerramento para um ponteiro para uma estrutura contendo um __unsigned char__  array e, em seguida, desreferenciar o ponteiro para que o compilador C coloque uma cópia dos argumentos na pilha antes de chamar a função no encerramento.

Para facilitar a criação da __execClosure()__ função, criaremos uma macro que facilita a criação dos vários tamanhos de structs que precisamos.
```
// helper macro to reduce type and reduce chance of typing errors.

#define CLOSEURESIZE(p,n)  if ((p)->nBytes < (n)) { \
struct {\
unsigned char x[n];\
} *px = (void *)p->args;\
p->pf(*px);\
}
```

Em seguida, usamos essa macro para criar uma série de testes para determinar como chamar a função de fechamento. Os tamanhos escolhidos aqui podem precisar de ajustes para aplicações específicas. Esses tamanhos são arbitrários e, como os dados de fechamento raramente terão o mesmo tamanho, isso não é eficiente usando o espaço da pilha. E existe a possibilidade de haver mais dados de fechamento do que permitimos.

```
// execute a closure by calling the function through the function pointer
// provided along with the created list of arguments.
ClosureStruct * execClosure(ClosureStruct *p)
{
    if (p) {
        // the following structs are used to allocate a specified size of
        // memory on the stack which is then filled with a copy of the
        // function argument list provided in the closure data.
        CLOSEURESIZE(p,64)
        else CLOSEURESIZE(p, 128)
        else CLOSEURESIZE(p, 256)
        else CLOSEURESIZE(p, 512)
        else CLOSEURESIZE(p, 1024)
        else CLOSEURESIZE(p, 1536)
        else CLOSEURESIZE(p, 2048)
    }

    return p;
}
```

Retornamos o ponteiro para o fechamento para torná-lo facilmente disponível.


# Um exemplo usando a biblioteca desenvolvida

Podemos usar o acima da seguinte forma. Primeiro, algumas funções de exemplo que realmente não fazem muito.

```
int zFunc(int i, int j, int k)
{
    printf("zFunc i = %d, j = %d, k = %d\n", i, j, k);
    return i + j + k;
}

typedef struct { char xx[24]; } thing1;

int z2func(thing1 a, int i)
{
    printf("i = %d, %s\n", i, a.xx);
    return 0;
}
```

Em seguida, construímos nossos encerramentos e os executamos.

```
{
    unsigned char closurearea[256];
    thing1 xpxp = { "1234567890123" };
    thing1 *ypyp = &xpxp;
    int  iValue = 45;

    ClosureStruct *dd = PUSHDATA(beginClosure(z2func, 256, malloc(256)), xpxp);
    free(execClosure(PUSHDATA(dd, iValue)));

    dd = PUSHDATA(beginClosure(z2func, 256, closurearea), *ypyp);
    dd = PUSHDATA(dd, 68);
    execClosure(dd);

    dd = PUSHDATA(beginClosure(zFunc, 256, closurearea), iValue);
    dd = PUSHDATA(dd, 145);
    dd = PUSHDATA(dd, 185);
    execClosure(dd);
}
```

O que dá uma saída de

```
i = 45, 1234567890123
i = 68, 1234567890123
zFunc i = 45, j = 145, k = 185
```

# Bem, e quanto ao Curry?
Em seguida, poderíamos fazer uma modificação em nossa estrutura de fechamento para nos permitir fazer curry de funções.

```
typedef struct {
    size_t  nBytes;    // current number of bytes of data
    size_t  nSize;     // maximum size of the data area
    size_t  nCurry;    // last saved nBytes for curry and additional arguments
    void(*pf)();       // pointer to the function to invoke
    unsigned char args[1];   // beginning of the data area for function arguments
} ClosureStruct;
```
com as funções de suporte para curry e redefinição de um ponto de curry sendo

```
ClosureStruct *curryClosure(ClosureStruct *p)
{
    p->nCurry = p->nBytes;
    return p;
}
ClosureStruct *resetCurryClosure(ClosureStruct *p)
{
    p->nBytes = p->nCurry;
    return p;
}
```
O código-fonte para testar isso pode ser:

```
{
    unsigned char closurearea[256];
    thing1 xpxp = { "1234567890123" };
    thing1 *ypyp = &xpxp;
    int  iValue = 45;

    ClosureStruct *dd = PUSHDATA(beginClosure(z2func, 256, malloc(256)), xpxp);
    free(execClosure(PUSHDATA(dd, iValue)));
    dd = PUSHDATA(beginClosure(z2func, 256, closurearea), *ypyp);
    dd = PUSHDATA(dd, 68);
    execClosure(dd);
    dd = PUSHDATA(beginClosure(zFunc, 256, closurearea), iValue);
    dd = PUSHDATA(dd, 145);
    dd = curryClosure(dd);
    dd = resetCurryClosure(execClosure(PUSHDATA(dd, 185)));
    dd = resetCurryClosure(execClosure(PUSHDATA(dd, 295)));
}
```
com a saída de
```
i = 45, 1234567890123
i = 68, 1234567890123
zFunc i = 45, j = 145, k = 185
zFunc i = 45, j = 145, k = 295
```

---

O ANSI C não possui suporte para encerramento, bem como funções aninhadas. A solução alternativa para isso é o uso de "struct" simples.

Fechamento de exemplo simples para soma de dois números.

```
// Structure for keep pointer for function and first parameter
typedef struct _closure{
    char* (*call)(struct _closure *str, int y);
    int x;
} closure;


// An function return a result call a closure as string
char *
sumY(closure *_closure, int y) {
    char *msg = calloc(20, sizeof(char));
    int sum = _closure->x + y;
    sprintf(msg, "%d + %d = %d", _closure->x, y, sum);
    return msg;
}


// An function return a closure for sum two numbers
closure *
sumX(int x) {
    closure *func = (closure*)malloc(sizeof(closure));
    func->x = x;
    func->call = sumY;
    return func;
}

int main (int argv, char **argc)
{

    closure *sumBy10 = sumX(10);
    puts(sumBy10->call(sumBy10, 1));
    puts(sumBy10->call(sumBy10, 3));
    puts(sumBy10->call(sumBy10, 2));
    puts(sumBy10->call(sumBy10, 4));
    puts(sumBy10->call(sumBy10, 5));
}
```  
## REFERENCE
[Is there a a way to achieve closures in C](https://stackoverflow.com/questions/4393716/is-there-a-a-way-to-achieve-closures-in-c)
