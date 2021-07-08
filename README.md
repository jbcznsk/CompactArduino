# Compact Arduino
## Como diminuir o tamanho (em bytes) do seu código com alguns truques simples

### Objetivo
As vezes, por necessidade ou curiosidade, queremos otimizar as coisas. Esse tutorial foca nisso, um jeito rápido para otimizar códigos para o Arduino (aqui será usado o UNO, mas pode ser adaptado para qualquer outro tomando os cuidados necessários).

### Componentes Utilizados
- Arduino UNO
- Arduino IDE

### Projeto
Como você escreveria um programa para simplesmente piscar o LED imbutido no seu Arduino UNO? Eu tenho 99% de certeza que você pensou em algo muito parecido (se não idêntico) ao abaixo:

```c
const int led = 13;        // Define o pino do LED

void setup(){
  pinMode(led, OUTPUT);    // Declara o pino como saída
}

void loop(){
  digitalWrite(led, HIGH); // Liga o LED
  delay(1000);             // Delay de 1 segundo
  digitalWrite(led, LOW);  // Desliga o LED
  delay(1000);             // Delay de 1 segundo
}
```
Ao compilarmos o código acima através da Arduino IDE para o Arduino UNO, temos os seguintes resultados:
```
Memória de Código: 924 bytes (2%)
Memória Dinâmica :   9 bytes (0%)
```
O que não é nada mal! Mas claro, esse é um programa extremamente simples, será que não podemos fazer melhor?

A primeira coisa que você pode pensar em fazer para reduzir o tamanho do código é trocar a variável ```const ìnt led``` por um ```#define```. O novo código ficaria assim: 
```c
#define LED  13            // Define o pino do LED

void setup(){
  pinMode(led, OUTPUT);    // Declara o pino como saída
}

void loop(){
  digitalWrite(LED, HIGH); // Liga o LED
  delay(1000);             // Delay de 1 segundo
  digitalWrite(LED, LOW);  // Desliga o LED
  delay(1000);             // Delay de 1 segundo
}
```
Eeeee... 
```
Memória de Código: 924 bytes (2%)
Memória Dinâmica :   9 bytes (0%)
```
Ué? Por que mesmo tirando uma variável e substituindo por uma constante não mudou em nada o tamanho?
Isso é porque o compilador já percebeu que essa variável não muda, portanto ele já fez essa substituição por trás dos panos, por isso os dois códigos ficaram do mesmo tamanho.

Agora vamos realmente fazer algumas mudanças que farão um bom impacto no tamanho do nosso código. Para isso, nós vamos acessar diretamente os registradores que controlam as portas do Arduino UNO.
Os registradores que controlam o led imbutido na placa são o conjunto B, o DDRB, PINB e PORTB.
| DDRB          | PINB          | PORTB  |
| ------------- |:-------------:| ----- |
| Data Direction Register | Input Pins Address | Data Register |

O primeiro uso será do DDRB. Vamos definir o pino 13 (PB5, como visto no datasheed do ATMega328p) como saída, ou seja, setar o quinto bit do registrador DDRB, e isso é feito (de forma elegante) através do operador de shiftar bits.
```c
#define LED  13            // Define o pino do LED

void setup(){
  DDRB |= (1 << 5);        // Declara o pino como saída
}

void loop(){
  digitalWrite(LED, HIGH); // Liga o LED
  delay(1000);             // Delay de 1 segundo
  digitalWrite(LED, LOW);  // Desliga o LED
  delay(1000);             // Delay de 1 segundo
}
```
E só fazendo isso já conseguimos economizar uma boa quantidade de bytes
```
Memória de Código: 860 bytes (2%)
Memória Dinâmica :   9 bytes (0%)
```
Agora o próximo passo é usar o PORTB para controlar o LED ao invés do digitalWrite.
```c
void setup(){
  DDRB |= (1 << 5);        // Declara o pino como saída
}

void loop(){
  PORTB |=  (1 << 5);      // Liga o LED
  delay(1000);             // Delay de 1 segundo
  PORTB &= ~(1 << 5);      // Desliga o LED
  delay(1000);             // Delay de 1 segundo
}
```
```
Memória de Código: 640 bytes (1%)
Memória Dinâmica :   9 bytes (0%)
```
Ou, se você quer deixar o código mais enxuto
```c
void setup(){
  DDRB |= (1 << 5);        // Declara o pino como saída
}

void loop(){
  PORTB ^= (1 << 5);       // Permuta o LED
  delay(1000);             // Delay de 1 segundo
}
```
E ai temos: 
```
Memória de Código: 602 bytes (1%)
Memória Dinâmica :   9 bytes (0%)
```
 Apenas utilizando diretamente os registradores ao invés das funções genéricas, já conseguimos economizar 322 bytes! Mas ainda tem mais, a função delay pode ser substituida por uma versão de mais baixo nível para deixar o código ainda menor.
 ```c
#include <avr/io.h>
#include "util/delay.h"

void setup(){
  DDRB |= (1 << 5);        // Declara o pino como saída
}

void loop(){
  PORTB ^= (1 << 5);       // Permuta o LED
  _delay_ms(1000);         // Delay de 1 segundo
}
```
E apenas com essa mudança nós conseguimos economizar 128 bytes!
```
Memória de Código: 474 bytes (1%)
Memória Dinâmica :   9 bytes (0%)
```

Por fim, também podemos mudar a estrutura do código. Sabemos que a função ```setup``` roda apenas uma vez, enquanto a função ```loop``` roda depois da ```setup``` e 
roda "eternamente". Portanto, se pensarmos no código com apenas uma função, temos o seguinte:

```c
#include <avr/io.h>
#include "util/delay.h"

int main(){
  DDRB |= (1 << 5);          // Declara o pino como saída

  while(1){
    PORTB ^= (1 << 5);       // Permuta o LED
    _delay_ms(1000);         // Delay de 1 segundo
  }
}
```
E fazemos essa alteração temos uma mudança gigantesca, como pode ser constatado abaixo
```
Memória de Código: 162 bytes (0%)
Memória Dinâmica :   0 bytes (0%)
```

### Resultados
Como pode ser visto, com apenas pequenas mudanças o código foi de 924 bytes para 162 bytes, uma mudança mais do que significativa.

### Conclusão
É claro que esse é um código simples de adaptar, em código maiores e mais complexos a dificuldade é maior, mas ainda sim é possível. Fazendo essas mudanças e diminuindo o tamanho do código, mais funções podem ser inseridas no mesmo microcontrolador, o que pode evitar o uso de um modelos mais caros para tarefas que podiam tranquilamente ser feitas por um modelo mais básico.

### Referências
- https://www.embarcados.com.br/atmel-studio/
