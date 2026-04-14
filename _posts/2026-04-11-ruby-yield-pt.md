---
layout: post
title: "Por que blocos em Ruby são mal compreendidos?"
date: 2026-04-11 10:00:00
lang: pt
permalink: /posts/ruby-yield-pt/
categories: [ruby, fundamentos]
tags: [ruby, yield, closures]
---

yield convenhamos é uma palavra que traduz de forma pobre para outras linguas. A tradução mais proxima que eu consegui pensar foi "ceder" o que não passa uma ideia boa do conceito. Na minha carreira eu ja tive oportunidade de trabalhar com programadores de diferentes niveis de senioridade e algo que me impressionou foi que o uso de blocos em ruby nem sempre é aplicado em todo o seu potencial.
Esse post é minha tentativa de desmistificar esse tema de uma vez.

## Por que voce esquece de usar blocos em suas soluções?

Isso se deve ao fato de pensarmos em métodos como caixas pretas, uma herança do modelo mental de C ou Java(antigo), onde tudo que é conhecido precisa ser passado por parâmetro. Em outras palavras voce não teve contato bastante com o conceito de closures, um dos conceitos mais elegantes, relevantes e poderoso em programação.

O modelo mental de closures não casa bem com o modelo mental de objetos e substantivos, ele te força a pensar em verbos e processos.

Sempre que você se pegar escrevendo o mesmo boilerplate em vários lugares (ex: abrir conexão, fazer algo, fechar conexão), pare. Crie um método que use `yield` e transforme o "fazer algo" em um bloco.

Exemplo ao invés de fazer isso:
```ruby
db = Conexao.conectar
db.query("...")
db.desconectar
```

Faça isso:
```ruby
def com_banco_de_dados
  db = Conexao.conectar
  yield(db)
ensure
  db.desconectar
end

# Uso:
com_banco_de_dados { |db| db.query("...") }
```

Pense no yield como uma forma de expressar:
"eu sei fazer algo mas deixo para voce(quem chamou o metodo) implementar os detalhes"

Esse conceito é simples e poderoso, tanto que o próprio Ruby o utiliza exaustivamente, como em `File.open { ... }` onde voce abre um arquivo entrega para voce e garante que ele feche depois.

## Blocos são a implementacao em ruby de closures
Em linguagens que não suportam closures de forma nativa ou simples (como o C puro), uma função é como um trabalhista com amnésia. Ela recebe ordens (parâmetros), executa e vai embora. Se ela precisar de uma informação que não foi passada no "contrato" (parâmetros), ela simplesmente não consegue acessar, a menos que a informação seja **Global** (o que é perigoso).

O problema fundamental que as closures resolvem é a **Preservação de Contexto sem Poluição Global**.
Essa frase é uma forma rebuscada de dizer que o seu código ganha uma "mochila particular" para carregar seus segredos.

Em termos simples:

  - Preservação de Contexto: Significa que o bloco não sofre daquela "amnésia" das funções comuns; ele lembra e carrega as variáveis de onde ele nasceu, não importa para onde ele viaje no seu programa.

  - Sem Poluição Global: Significa que ele faz isso de forma privada. Ele não precisa gritar essas informações para todo o sistema ouvir (criando variáveis globais), o que evita que outros processos acabem "tropeçando" ou quebrando seus dados por acidente.

Basicamente, é a diferença entre anotar um segredo em um diário que você leva no bolso (Closure) versus escrevê-lo em um outdoor no meio da cidade (Global) só para não esquecer.
Em termos técnicos, dizemos que a closure se 'fecha sobre' o seu escopo léxico, garantindo que essas variáveis permaneçam vivas enquanto o bloco precisar delas. É por isso que, no Ruby, o bloco nunca sofre de amnésia: ele carrega consigo o DNA do lugar onde nasceu.

### O problema do escopo Mudo
Sem closures, para levar um dado de um ponto A para um ponto B dentro de uma lógica complexa, você teria que:

1. Passar o dado por todos os métodos intermediários (mesmo que eles não o usem).
2. Ou criar uma variável global (que todo mundo pode quebrar).

A closure resolve isso "congelando" o ambiente ao redor do código. Ela não é apenas uma lista de instruções; ela é um **objeto que carrega o seu lugar de origem**.

### O que é o yield?
yield é uma instrução direta para a VM que cria um ponteiro para o contexto onde o bloco foi chamado. Em outras palavras: em Ruby o yield olha para um ponteiro chamado EP (Environment Pointer). Esse ponteiro diz a VM: "Ei, se o codigo do bloco pedir pela variavel x e ela nao estiver aqui dentro, olhe nesse endereço de memoria aqui, que era onde o bloco nasceu".

Por isso que isto funciona:
```ruby
nome = "Café"
3.times do
  puts "hora do #{nome}"
end
```

### Para que serve o yield?

| **Função**                    | **O que faz**                                                                                          | **Exemplo Prático**                  |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| **Encapsulamento** | Permite que variáveis vivam além do fim da execução de um escopo, mas permaneçam protegidas de acesso externo. | Criar contadores privados sem usar classes. |
| **Avaliação Preguiçosa** | Você define _o que_ fazer agora, mas decide _quando_ executar depois, mantendo acesso aos dados originais. | Callbacks de eventos ou requisições HTTP. |
| **Injeção de Comportamento** | O método define a infraestrutura (`start_time`, `end_time`), e a closure fornece o comportamento. | Um benchmark ou medição de tempo de execução |

Nota: Nem sempre quem usa seu método enviará um bloco. Para evitar que seu código "exploda" com um `LocalJumpError`, o Ruby oferece o método `block_given?`. Ele é o porteiro que verifica se há uma "estratégia" antes de tentar ceder o controle.

Exemplo:
```ruby
def executar_comando
  return puts "Aviso: Nenhum comando foi enviado para execução." unless block_given?
  
  puts "Iniciando sistema..."
  yield
  puts "Sistema finalizado."
end

# Uso seguro (sem bloco):
executar_comando
# Saída: Aviso: Nenhum comando foi enviado para execução.

# Uso padrão (com bloco):
executar_comando { puts "Executando: Limpeza de Cache" }
# Saída: 
# Iniciando sistema...
# Executando: Limpeza de Cache
# Sistema finalizado.
```

### Yield é um padrão de projeto simplificado
O yield é o padrao de projeto Strategy implementado nativamente na linguagem.
Isso mesmo, para fazer o que o yield faz em Java (antes do Java8) voce precisaria criar um interface, uma classe que implementasse essa interface e instanciar um objeto.
Ruby resolve isso de forma muito elegante trazendo para dentro da linguagem a capacidade de um metodo ceder(yield) o controle para quem o chamou definir os detalhes(estrategias) que quiser.

### Yield é uma transferencia direta de fluxo dentro da VM

Ao contrário do que muitos pensam, o `yield` não cria um objeto na memória para funcionar. Enquanto o uso de parâmetros como `&block` transforma o bloco em uma instância da classe `Proc` (o que custa memória e processamento), o `yield` é uma instrução de "pulo" direto na **YARV** (a Máquina Virtual do Ruby).

Isso significa que:

1. **Performance Cirúrgica:** O Ruby apenas pausa o método atual e salta para o endereço de memória do bloco. Não há criação de objetos pesados.
2. **Eficiência de Escopo:** Graças ao **EP (Environment Pointer)**, o bloco acessa as variáveis de fora sem precisar que o Ruby faça cópias de dados. Ele apenas aponta para onde elas já estão.


## Conclusão
O `yield` é o segredo da elegância do Ruby. Ele transforma métodos em **plataformas**, onde você fornece a estrutura e deixa o espaço em branco para o usuário preencher.

Ao entender que ele é uma **Closure** (código + contexto), você para de lutar contra a linguagem e começa a escrever códigos que são, ao mesmo tempo, flexíveis e protegidos. Na próxima vez que vir um padrão repetitivo (abrir/fechar, iniciar/parar), não crie um método rígido: **ceda o controle**.
