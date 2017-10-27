# Desenvolvendo uma versão mocada

Agora que temos uma boa estrutura preparada, vamos começar a desenvolver nossa API.
Um dos guideline arquiteturais mais falados hoje em dia é o
[Onion Architecture](http://jeffreypalermo.com/blog/the-onion-architecture-part-1/),
que preza por deixar o domínio no centro e a infra de storage/apresentação nas beiradas,
em oposição a arquiteturas comuns 3 camadas que deixam tudo na vertical. Utilizaremos
esse estilo arquitetural durante a nossa trilha.

Daqui para frente falaremos sempre sobre três camadas lógicas da nossa aplicação:
Domínio, Apresentação e Repositório. Espera-se que o leitor correlacione essas camadas
com o estilo arquitetural *Onion Architecture*, principalmente as seguintes
propriedades:

* **Domínio** é a camada central, não deve depender de nenhuma biblioteca de infraestrutura
(isso inclui drivers de conexão com banco, APIs HTTP etc). Essa camada expõe as ideias
necessárias para fazer nossa solução funcionar.
* **Apresentação** é uma camada exterior, que pode depender apenas do domínio. Ela é
responsável principalmente por mapear endpoints HTTP a serviços do domínio.
* **Repositório** é também uma camada exterior, que pode depender apenas do domínio.
Na maior parte dos casos ela contém *implementações concretas* de interfaces de
repositório criadas no domínio da aplicação, conectando essas ideias com uma
infraestrutura de persistência.

Um outro ponto que é importante estabelecer previamente é com relação ao agrupamento
das nossas classes. Existem basicamente duas maneiras de tratar o empacotamento de
tipos em projetos Java:

1. Baseado no *tipo ou papel técnico* da classe:

```
com.projeto
\_.controllers
     FilmesController
     CurtidasController
     UsuariosController
\_.services
     FilmesService
     CurtidasService
     UsuariosService
\_.repositories
     FilmesRepository
     CurtidasRepository
     UsuariosRepository
\_.entities
     Filme
     Curtida
     Usuario
\_.dto
     ParametrosDePaginacao
     PaginaDeRegistros<T>
```

2. Baseado no *módulo de negócio* a que o tipo pertence:

```
com.projeto
\_.filmes
     FilmesController
     FilmesService
     FilmesRepository
     Filme
\_.curtidas
     CurtidasController
     CurtidasService
     CurtidasRepository
     Curtida
\_.usuarios
     UsuariosController
     UsuariosService
     UsuariosRepository
     Usuario
\_.comum
     ParametrosDePaginacao
     PaginaDeRegistros<T>
```

Optaremos pela segunda opção, por dois motivos:

1. Fica mais fácil encontrar classes de contexto: as chances são grandes de que em
uma tarefa você vai impactar várias classes relacionadas ao mesmo módulo de negócio,
e o fato delas estarem todas agrupadas facilita um pouco;
2. Uma análise estática de dependência entre os pacotes gera informação muito mais
valiosa com esse empacotamento (por exemplo, você pode aferir quem depende de `comum`
e que `comum` não depende de mais ninguém), o que não ocorre com o outro estilo.

Vamos então começar o desenvolvimento criando a classe `Filme`, no pacote
`com.opensanca.trilharest.filmes.filmes` (o primeiro `filmes` é o sistema, o segundo
é o pacote de negócio):

```java
public class Filme {

  private UUID id;
  private String nome;
  private String sinopse;
  private Duration duracao;
  private LocalDate inicioExibicao;
  private LocalDate fimExibicao;

  // getters e setters foram omitidos
}

```

UUID? Hoje em dia é considerado boa prática o uso de identificadores globais únicos,
os UUIDs ou GUIDs, como identificadores de registros (existe grande debate sobre
os prós e contras de cada abordagem no entanto¹²). Um dos principais benefícios é
não depender do banco de dados para gerar um identificador único, propriedade que
facilita muito o desenvolvimento de aplicações distribuídas, até mesmo com múltiplos
storages distintos (como uma base relacional e nosql que compartilham registros
entre si).

¹ https://stackoverflow.com/questions/45399/advantages-and-disadvantages-of-guid-uuid-database-keys

² https://tomharrisonjr.com/uuid-or-guid-as-primary-keys-be-careful-7b2aa3dcb439

Duration, LocalDate? O Java 8 trouxe uma API de datas totalmente reformulada, e é
de nosso total interesse usá-la sempre que possível. Como veremos mais pra frente isso
não vem de graça, pois temos que lidar com suporte recente dentro de bibliotecas como
o Spring MVC, mas no final das contas vale a pena. Caso o leitor desconheça a API de
datas do Java 8, sugiro as seguintes leituras¹².

¹ http://blog.caelum.com.br/conheca-a-nova-api-de-datas-do-java-8/

² http://www.baeldung.com/java-8-date-time-intro

Note que a classe `Filme` faz parte do `domínio` da nossa aplicação, pois ela representa
uma ideia/conceito necessário para seu funcionamento. Vamos agora adicionar no nosso
domínio a capacidade de *obter os filmes em exibição* e *obter detalhes de um filme
pelo seu identificador*. Para isso, criaremos uma interface chamada *FilmesRepository*
no pacote `com.opensanca.trilharest.filmes.filmes`:

```java
import java.util.UUID;

public interface FilmesRepository {

  Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao);

  Filme buscarPorId(UUID id);

}
```

Note que esse tipo faz parte do **domínio** e não da camada **repositório**, como o nome
pode deixar transparecer.

Repare que esse tipo depende de dois conceitos de negócio que ainda não temos na solução:
`Pagina` e `ParametrosDePaginacao`. Vamos criar esses modelos no pacote
`com.opensanca.trilharest.filmes.comum`:

```java
import java.util.List;

public class Pagina<T> {

  private List<T> registros;
  private Integer totalDeRegistros;

  // getters e setters
}
```

```java
public class ParametrosDePaginacao {

  private Integer pagina;
  private Integer tamanhoDaPagina;

  // getters e setters
}
```

Volte agora na classe `FilmesRepository` e importe os modelos criados acima.

Repare que, consultando nosso estilo arquitetural, já deve ser possível implementar
a API REST, mesmo sabendo que não existe implementação concreta para o repositório
de filmes. Obviamente que a aplicação não deve funcionar, mas pelo menos deve compilar,
pois a camada de apresentação não pode depender diretamente da camada de repositório.
Para validar isso, vamos tentar implementar nossa API e ver até onde conseguimos chegar.
Crie uma classe chamada `FilmesAPI` no pacote `com.opensanca.trilharest.filmes.filmes`:

```java
import com.opensanca.trilharest.filmes.comum.Pagina;
import com.opensanca.trilharest.filmes.comum.ParametrosDePaginacao;
import java.util.UUID;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/filmes")
public class FilmesAPI {

  @Autowired
  private FilmesRepository filmesRepository;

  @GetMapping("/em-exibicao")
  public Pagina<Filme> get(ParametrosDePaginacao parametrosDePaginacao) {
    return filmesRepository.buscarPaginaEmExibicao(parametrosDePaginacao);
  }

  @GetMapping("/{id}")
  public Filme getPorId(@PathVariable UUID id) {
    return filmesRepository.buscarPorId(id);
  }

}
```

Observações:

* `@GetMapping` é um "atalho/apelido/alias" para
`@RequestMapping(method={RequestMethod.GET})`. Existem correspondentes para os
outros métodos HTTP também, como `@PostMapping` e `@DeleteMapping`.
* `@PathVariable` diz para o Spring MVC que queremos que ele passe neste parâmetro
o valor de uma *variável de caminho/URL*. Essas variáveis são os termos que estão
entre chaves no mapeamento da requisição em questão. Neste exemplo, este
mapeamento é `/filmes/{id}`, e ele contém uma única variável de caminho: `id`.
Note que se não passarmos nada para o parâmetro `value` da anotação, o Spring MVC vai
usar o nome do parâmetro para decidir qual variável de caminho buscar.

Em outras palavras, ao submeter uma requisição `GET /filmes/123456`, o Spring MVC vai
capturar a expressão `123456`, tentar converter para o tipo do parâmetro `id` na action
e passar o valor resultante ao chamar a mesma.

Se tentarmos executar nossa aplicação neste momento veremos que ela dispara o seguinte
erro:

```
Description:

Field filmesRepository in com.opensanca.trilharest.filmes.filmes.FilmesAPI required a bean of type 'com.opensanca.trilharest.filmes.filmes.FilmesRepository' that could not be found.


Action:

Consider defining a bean of type 'com.opensanca.trilharest.filmes.filmes.FilmesRepository' in your configuration.
```

Veja que o próprio Spring nos dá a dica do que precisamos fazer, que é disponibilizar
para o contexto de injeção de dependência uma implementação da `ideia` abstrata
`FilmesRepository`.

Vamos fazer isso implementando um repositório em memória (buscando registros
em um `java.util.ArrayList` estático). No futuro mudaremos isso para um repositório
JPA/Hibernate (consumindo uma base de dados PostgreSQL).

Antes de criarmos a classe de repositório, vamos criar um construtor na classe `Filme`
que nos será útil para criar a massa de dados de teste:

```java
public Filme() {
}

public Filme(UUID id, String nome, String sinopse, Duration duracao,
    LocalDate inicioExibicao, LocalDate fimExibicao) {
  this.id = id;
  this.nome = nome;
  this.sinopse = sinopse;
  this.duracao = duracao;
  this.inicioExibicao = inicioExibicao;
  this.fimExibicao = fimExibicao;
}
```

Note que também criamos um construtor default, pois ele será
necessário no futuro quando integrarmos o Spring Data no projeto.

Agora crie uma classe chamada `FilmesRepositoryRAM` no pacote
`com.opensanca.trilharest.filmes.filmes`:

```java
import com.opensanca.trilharest.filmes.comum.Pagina;
import com.opensanca.trilharest.filmes.comum.ParametrosDePaginacao;
import java.time.Duration;
import java.time.LocalDate;
import java.util.Arrays;
import java.util.List;
import java.util.UUID;
import org.springframework.stereotype.Repository;

@Repository
public class FilmesRepositoryRAM implements FilmesRepository {

  private static List<Filme> registros = Arrays.asList(
      new Filme(UUID.randomUUID(), "Filme 1", "Sinopse do filme 1",
          Duration.ofMinutes(153),
          LocalDate.of(2017, 10, 1),
          LocalDate.of(2017, 10, 25)),
      new Filme(UUID.randomUUID(), "Filme 2", null,
          null,
          LocalDate.of(2017, 10, 2),
          LocalDate.of(2017, 10, 24)),
      new Filme(UUID.randomUUID(), "Filme 3", null,
          null, null, null),
      new Filme(UUID.randomUUID(), "Filme 4", "Sinopse do filme 4",
          Duration.ofHours(1),
          LocalDate.of(2016, 10, 2),
          LocalDate.of(2016, 10, 19))
  );

  @Override
  public Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao) {
    return null;
  }

  @Override
  public Filme buscarPorId(UUID id) {
    return null;
  }

}
```

Note que até agora só preparamos uma massa de dados simulada, em memória, de filmes.
Para melhorar a qualidade dessa massa de testes, garanta que os filmes 1 e 2 estão
com as datas de início e fim de exibição configuradas de modo a *estarem em exibição*
relativamente à data atual, seja ela qual for agora no momento da leitura desse
material.

Vamos começar a implementação pelo método `buscarPorId`. Primeiro com uma versão
imperativa:

```java
@Override
public Filme buscarPorId(UUID id) {
  for (int i = 0; i < registros.size(); i++) {
    if (registros.get(i).getId().equals(id)) {
      return registros.get(i);
    }
  }
  return null;
}
```

Pare um momento e reflita sobre esse código. O que aquele `return null` faz ali? Sem
dúvida que é uma implementação aceitável, mas onde documentamos que seria esse o
retorno do método? Por quê não disparar uma exceção? Não existe resposta correta para
nenhuma dessas perguntas, mas uma coisa é certa, quando uma assinatura de método
não é suficiente para deixar claro o que um método faz, deveríamos estar documentando-o
melhor. Veja como podemos fazer isso, na interface `FilmesRepository`:

```java
/**
  * @return o filme correspondente ao identificador passado
  * como parâmetro ou null se não encontrado.
  */
Filme buscarPorId(UUID id);
```

Bem melhor, mas podemos também optar por disparar uma exceção neste cenário:

```java
@Override
public Filme buscarPorId(UUID id) {
  for (int i = 0; i < registros.size(); i++) {
    if (registros.get(i).getId().equals(id)) {
      return registros.get(i);
    }
  }
  throw new IllegalArgumentException("Filme não encontrado!");
}
```

E claro, documentar apropriadamente:

```java
/**
  * @throws IllegalArgumentException se o filme não for encontrado
  */
Filme buscarPorId(UUID id);
```

Usar uma exceção como a `IllegalArgumentException` é claramente melhor do que usar
uma exceção genérica como `Exception` ou `RuntimeException`, mas nada melhor do que
criar exceções específicas em seu domínio. Vamos criar uma classe de exceção
chamada `EntidadeNaoEncontradaException` no pacote
`com.opensanca.trilharest.filmes.comum`:

```java
public class EntidadeNaoEncontradaException extends RuntimeException {
}
```

Poderíamos ter estendido `Exception` ao invés de `RuntimeException`, mas hoje em dia
se discute se exceções verificadas são realmente boas ou se são uma má prática¹²³.

¹ https://www.javaworld.com/article/3142626/core-java/are-checked-exceptions-good-or-bad.html

² https://stackoverflow.com/questions/613954/the-case-against-checked-exceptions

³ https://docs.oracle.com/javase/tutorial/essential/exceptions/runtime.html

Neste caso como queremos que esse tipo de exceção seja tratada de maneira genérica,
nós não faremos delas exceções verificadas.

Uma vez criada a exceção, podemos substituir o uso de `IllegalArgumentException`:

```java
@Override
public Filme buscarPorId(UUID id) {
  for (int i = 0; i < registros.size(); i++) {
    if (registros.get(i).getId().equals(id)) {
      return registros.get(i);
    }
  }
  throw new EntidadeNaoEncontradaException();
}
```

E atualizar a documentação:

```java
/**
  * @throws EntidadeNaoEncontradaException
  */
Filme buscarPorId(UUID id);
```

Bem melhor. Podemos ainda refatorar nossa solução imperativa para uma solução funcional,
usando a API de Streams introduzida no Java 8:

```java
@Override
public Filme buscarPorId(UUID id) {
  return registros.stream()
      .filter(x -> x.getId().equals(id))
      .findFirst()
      .orElseThrow(() -> new EntidadeNaoEncontradaException());
}
```

Para um problema tão simples é discutível se vale a pena usar uma solução funcional,
de fato ela não parece ajudar muito na legibilidade tampouco na performance do código.

Vamos agora implementar o método `buscarPaginaEmExibicao`:

```java
@Override
public Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao) {

  List<Filme> emExibicao = registros.stream()
      .filter(filme -> filme.emExibicao())
      .collect(Collectors.toList());

  Integer pagina = parametrosDePaginacao.getPagina();
  Integer tamanhoDaPagina = parametrosDePaginacao.getTamanhoDaPagina();
  Integer primeiroRegistro = (pagina - 1) * tamanhoDaPagina;
  Integer ultimoRegistro = primeiroRegistro + tamanhoDaPagina;
  Integer totalDeRegistros = emExibicao.size();
  List<Filme> registros = emExibicao.subList(primeiroRegistro,
      Math.min(totalDeRegistros, ultimoRegistro));

  Pagina<Filme> paginaDeRegistros = new Pagina<>();
  paginaDeRegistros.setRegistros(registros);
  paginaDeRegistros.setTotalDeRegistros(totalDeRegistros);
  return paginaDeRegistros;
}
```

Note que estamos usando um método chamado `emExibicao` que não existe na classe `Filme`.
Vamos criá-lo mas sem implementá-lo corretamente ainda:

```java
public boolean emExibicao() {
  return true;
}
```

Por quê não implementá-lo ainda? Como estamos codificando métodos na nossa camada de
*domínio*, usaremos TDD (Test-Driven Development) durante todo o processo.

Repare que o seguinte bloco de código, extraído do método `buscarPaginaEmExibicao`
implementado mais acima, também pode ser visto como código de domínio, pois só
depende de dados do domínio (propriedades dos parâmetros de paginação):

```java
Integer pagina = parametrosDePaginacao.getPagina();
Integer tamanhoDaPagina = parametrosDePaginacao.getTamanhoDaPagina();
Integer primeiroRegistro = (pagina - 1) * tamanhoDaPagina;
Integer ultimoRegistro = primeiroRegistro + tamanhoDaPagina;
Integer totalDeRegistros = emExibicao.size();
```

Isso é verdade e foi feito assim de propósito, para tentar ilustrar ao leitor que às
vezes é muito fácil *vazar* implementações de domínio nas camadas externas de nossa
arquitetura, o que fica evidente quando nos deparamos em projetos cuja *camada de
negócio* nada mais é do que um conjunto de métodos de uma linha que transportam 
os dados da camada de apresentação para a camada de persistência. Evite isso! Temos
que aplicar força na direção de levar a maior quantidade de código para *dentro* da
camada de domínio, e não o contrário!

Note que neste momento nossa aplicação já está funcional, mesmo considerando a
implementação incorreta do método `emExibicao` na classe `Filme`. Suba a aplicação
e chame a seguinte URL no seu navegador:
`http://localhost:8080/filmes/em-exibicao?pagina=1&tamanhoDaPagina=3`. Se a resposta
foi parecida com essa, deu certo!

```json
{"registros":[{"id":"54fcd988-78c3-46a6-8bd0-8b8b73de3117","nome":"Filme 1","sinopse":"Sinopse do filme 1","duracao":{"seconds":9180,"nano":0,"negative":false,"zero":false,"units":["SECONDS","NANOS"]},"inicioExibicao":{"year":2017,"month":"OCTOBER","chronology":{"calendarType":"iso8601","id":"ISO"},"era":"CE","dayOfYear":274,"dayOfWeek":"SUNDAY","leapYear":false,"dayOfMonth":1,"monthValue":10},"fimExibicao":{"year":2017,"month":"OCTOBER","chronology":{"calendarType":"iso8601","id":"ISO"},"era":"CE","dayOfYear":293,"dayOfWeek":"FRIDAY","leapYear":false,"dayOfMonth":20,"monthValue":10}},{"id":"aa2b3313-134c-494a-ad4e-51d0a877ca7a","nome":"Filme 2","sinopse":null,"duracao":null,"inicioExibicao":{"year":2017,"month":"OCTOBER","chronology":{"calendarType":"iso8601","id":"ISO"},"era":"CE","dayOfYear":275,"dayOfWeek":"MONDAY","leapYear":false,"dayOfMonth":2,"monthValue":10},"fimExibicao":{"year":2017,"month":"OCTOBER","chronology":{"calendarType":"iso8601","id":"ISO"},"era":"CE","dayOfYear":292,"dayOfWeek":"THURSDAY","leapYear":false,"dayOfMonth":19,"monthValue":10}},{"id":"e7d6430d-e53c-4008-9c65-f8c7879b6974","nome":"Filme 3","sinopse":null,"duracao":null,"inicioExibicao":null,"fimExibicao":null}],"totalDeRegistros":4}
```

Desconsidere a horrível apresentação das informações, vamos ir melhorando isso de
pouco em pouco. Tente agora essa URL: `http://localhost:8080/filmes/em-exibicao`.
A aplicação vai dar um erro, pois o Spring MVC não criou o objeto
`parametrosDePaginacao`, visto que nenhuma propriedade dele foi passada como
parâmetro para a requisição. Seria interessante resolver isso, entregando a primeira
página com um tamanho padrão neste caso. Vamos resolver isso na classe `FilmesAPI`:

```java
@GetMapping("/em-exibicao")
public Pagina<Filme> get(ParametrosDePaginacao parametrosDePaginacao) {
  if (parametrosDePaginacao == null) {
    parametrosDePaginacao = new ParametrosDePaginacao();
  }
  return filmesRepository.buscarPaginaEmExibicao(parametrosDePaginacao);
}
```

E inicializando as propriedades com padrões sensíveis na classe `ParametrosDePaginacao`:

```java
private Integer pagina = 1;
private Integer tamanhoDaPagina = 3;
```

Reinicie a aplicação e tente novamente abrir a URL
`http://localhost:8080/filmes/em-exibicao`. Agora é para estar funcionando corretamente.

Vamos agora nos voltar ao método `emExibicao`. Como já falamos vamos implementar todos
os métodos de domínio usando a técnica [TDD](http://tdd.caelum.com.br/). Basicamente,
o TDD nos faz seguir o seguinte fluxo de trabalho:

1. **Testes passando e código refatorado**: escreva um teste que falha.
2. **Testes quebrando**: escreve o código mais simples possível que faça-os passarem.
3. **Testes passando**: refatore a solução adicionando boas práticas.

O principal desafio do TDD é essa parte: *código mais simples possível*. É nesta etapa
que acabamos escrevendo código demais, resolvendo problemas que ainda não estão cobertos
por testes unitários. Essa imagem reflete bem o que ocorre:

![How to Draw a Horse](https://cdn-images-1.medium.com/max/1600/1*mipae2XULpWaC2k_tcrYQA.jpeg)

Com isso em mente, considerando que neste momento estamos na situação **1** (testes
passando e código refatorado, afinal somos todos programadores e sabemos que zero testes podem ser classificados como "testes passando"), vamos criar nossa primeira suíte de
teste.

A principal biblioteca usada para a escrita de testes unitários em Java é a biblioteca
`JUnit`. Com ela criamos classes que agrupam vários *casos de teste*, onde cada caso de
teste é um método anotado com a anotação `@Test` do pacote `org.junit`. Não precisamos
adicionar uma dependência para o `JUnit` pois ela já está no nosso `build.gradle` desde
a criação do projeto. O que precisamos fazer, no entanto, é criar um diretório separado
para as classes de teste, para que estas não façam parte do pacote final da aplicação
(o que não faria nenhum sentido):

```
$ mkdir -p src/test/java
$ gradle idea
```

Agora vamos criar uma classe chamada `FilmeTest` no pacote
`com.opensanca.trilharest.filmes.filmes`, no diretório de fontes `src/test/java`:

```java
import org.junit.Assert;
import org.junit.Test;

public class FilmeTest {

  @Test
  public void foraDeExibicaoSeDatasNulas() {
    // preparação
    Filme filme = new Filme(null, null, null, null, null, null);
    // processamento
    boolean emExibicao = filme.emExibicao();
    // verificação
    Assert.assertFalse(emExibicao);
  }

}
```

Observações:

* Só faz sentido testar unitariamente os **métodos públicos** das suas classes. Afinal
os métodos privados de uma classe só devem existir se estiverem sendo usados por no
mínimo um método público, portanto testando a API pública da sua classe você deverá
ser capaz de testá-la por completo.
* É boa prática criar uma suíte de teste para cada classe que contenha no mínimo
um método público. Essa suíte normalmente possui o mesmo pacote e o mesmo nome
da classe testada, porém com o sufixo `Test`.
* Regras de nomenclatura de método normalmente não se aplicam da mesma forma para
métodos de teste unitário. Preferem-se métodos com nomes mais longos mas que consigam
deixar bem claro o cenário que está sendo analisado.

Repare que nosso caso de teste parece ter uma estrutura em seções bem definida:

* Seção **preparação**: aqui criamos instâncias e preparamos todo o estado necessário.
Em outras palavras preparamos a massa de dados do teste.
* Seção **processamento**: nessa parte executamos o bloco de código que estamos testando,
colhendo os resultados em variáveis para futura verificação.
* Seção **verificação**: aqui aplicamos uma ou mais *asserções* nos resultados da seção
de processamento, com o fim de analisar se o resultado é de fato o que esperávamos.

Rodando essa suíte de teste unitário (procure rapidamente como fazer isso na sua IDE,
normalmente a IDE já está integrada com o JUnit e uma opção para rodar a suíte aparece
se você clicar com o botão direito no projeto, na classe da suíte ou no método do
caso de teste) você verá que ela está falhando. Neste ponto chegamos na situação
**2** (testes quebrando). E temos que **implementar o código mais simples** que faz
os testes passarem. E esse seria:

```java
public boolean emExibicao() {
  return false;
}
```

Isso não é brincadeira, de fato essa facilidade com que conseguimos ludibriar nossa
suíte de teste só demonstra o quão fraca ela é! Agora que estamos na situação **3**
(testes passando), deveríamos refatorar nosso código, mas como não há nada a fazer
vamos direto para a situação **1** novamente. Neste ponto adicionaremos mais um
caso de teste, sempre pensando no cenário mais simples que ainda não cobrimos:

```java
@Test
public void emExibicaoSeDentroDeIntervaloDeDatas() {
  // preparação
  Filme filme = new Filme(null, null, null, null,
      LocalDate.of(2017, 10, 1),
      LocalDate.of(2017, 10, 30));
  // processamento
  boolean emExibicao = filme.emExibicao();
  // verificação
  Assert.assertTrue(emExibicao);
}
```

Agora que quebramos nossa suíte novamente, estamos na situação **2**. Vamos implementar
a solução mais simples:

```java
public boolean emExibicao() {
  return getInicioExibicao() != null;
}
```

Chegamos no passo **3**, e vamos direto para o passo **1** sem refatorar nada. O 
próximo cenário que trataremos será:

```java
@Test
public void foraDeExibicaoSeForaDoIntervaloDeDatas() {
  // preparação
  Filme filme = new Filme(null, null, null, null,
      LocalDate.of(2016, 10, 1),
      LocalDate.of(2016, 10, 30));
  // processamento
  boolean emExibicao = filme.emExibicao();
  // verificação
  Assert.assertFalse(emExibicao);
}
```

E agora a implementação:

```java
public boolean emExibicao() {
  if (getInicioExibicao() == null) {
    return false;
  }
  LocalDate hoje = LocalDate.now();
  return getInicioExibicao().isBefore(hoje) && getFimExibicao().isAfter(hoje);
}
```

Agora temos um problema aqui, o que ocorre com essa suíte de teste depois do dia
`31/10/2017` (fim do período de exibição do filme do caso de teste
`emExibicaoSeDentroDeIntervaloDeDatas`)? Ela quebra! Isso ocorre pois nosso domínio
está implementado de uma forma `não determinística`, ou seja, com os mesmos parâmetros
ele pode **retornar resultados diferentes**. Não queremos isso de modo algum na
camada de domínio! Como resolver? A resposta **não** é usar o `LocalDate.now()` também
nos testes unitários, pois isso tornariam os testes não determinísticos também, o que
é uma situação ainda mais complicada e não recomendada.

A solução correta é transformar nosso domínio em um domínio determinístico, dessa forma:

```java
public boolean emExibicao(LocalDate referencia) {
  if (getInicioExibicao() == null) {
    return false;
  }
  return getInicioExibicao().isBefore(referencia) && getFimExibicao().isAfter(referencia);
}
```

Note que teremos que ajustar os impactos causados por isso. Primeiro na classe
`FilmesRepositoryRAM`:

```java
@Override
public Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao, LocalDate referencia) {

  List<Filme> emExibicao = registros.stream()
      .filter(filme -> filme.emExibicao(referencia))
      .collect(Collectors.toList());

  [...]
```

Note como adicionamos um parâmetro no método `buscarPaginaEmExibicao` ao invés de usar
o `LocalDate.now()`. Fizemos isso pois esse código implementa uma interface de domínio
que também deve ser *determinística*. Ajuste agora a interface `FilmesRepository`:

```java
Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao, LocalDate referencia);
```

A classe `FilmesAPI`:

```java
@GetMapping("/em-exibicao")
public Pagina<Filme> get(ParametrosDePaginacao parametrosDePaginacao) {
  if (parametrosDePaginacao == null) {
    parametrosDePaginacao = new ParametrosDePaginacao();
  }
  LocalDate hoje = LocalDate.now();
  return filmesRepository.buscarPaginaEmExibicao(parametrosDePaginacao, hoje);
}
```

Repare que aqui sim podemos usar o não determinismo. Por fim falta ajustar todos 
os casos de teste de modo a passarem a data de referência:

```java
@Test
public void XXXXXXXXXXXXXXX() {
  // preparação
  Filme filme = XXXXXXXXXXXXXXXXXXXXXX;
  LocalDate referencia = LocalDate.of(2017, 10, 22);
  // processamento
  boolean emExibicao = filme.emExibicao(referencia);
  // verificação
  Assert.assertXXXXXXXXXXXXXXXXXX(emExibicao);
}
```

Note que agora mesmo voltando ao passado a suíte não vai quebrar.

Voltando ao método `emExibicao`, vamos para o próximo cenário não tratado:

```java
@Test
public void emExibicaoSeInicioExatamenteHoje() {
  // preparação
  Filme filme = new Filme(null, null, null, null,
      LocalDate.of(2017, 10, 22),
      LocalDate.of(2017, 10, 30));
  LocalDate referencia = LocalDate.of(2017, 10, 22);
  // processamento
  boolean emExibicao = filme.emExibicao(referencia);
  // verificação
  Assert.assertTrue(emExibicao);
}
```

E mais uma vez ajustar a implementação:

```java
public boolean emExibicao(LocalDate referencia) {
  if (getInicioExibicao() == null) {
    return false;
  }
  return (getInicioExibicao().isEqual(referencia) || getInicioExibicao().isBefore(referencia)) &&
      getFimExibicao().isAfter(referencia);
}
```

Note que dessa vez podemos refatorar o código para que fique mais claro:

```java
public boolean emExibicao(LocalDate referencia) {
  if (getInicioExibicao() == null) {
    return false;
  }
  LocalDate inicio = getInicioExibicao();
  boolean depoisDoInicio = inicio.isEqual(referencia) || inicio.isBefore(referencia);
  return depoisDoInicio && getFimExibicao().isAfter(referencia);
}
```

Vamos agora tratar o mesmo caso mas no término:

```java
@Test
public void emExibicaoSeTerminoExatamenteHoje() {
  // preparação
  Filme filme = new Filme(null, null, null, null,
      LocalDate.of(2017, 10, 20),
      LocalDate.of(2017, 10, 22));
  LocalDate referencia = LocalDate.of(2017, 10, 22);
  // processamento
  boolean emExibicao = filme.emExibicao(referencia);
  // verificação
  Assert.assertTrue(emExibicao);
}
```

```java
public boolean emExibicao(LocalDate referencia) {
  if (getInicioExibicao() == null) {
    return false;
  }
  LocalDate inicio = getInicioExibicao();
  LocalDate fim = getFimExibicao();
  boolean depoisDoInicio = inicio.isEqual(referencia) || inicio.isBefore(referencia);
  boolean antesDoFim = fim.isEqual(referencia) || fim.isAfter(referencia);
  return depoisDoInicio && antesDoFim;
}
```

E por fim o cenário onde o filme não possui uma data de fim de exibição (nossa regra
diz que os filmes sem data de início ou fim de exibição são considerados cadastros
incompletos e por isso não estão em exibição):

```java
@Test
public void foraDeExibicaoSeTerminoNulo() {
  // preparação
  Filme filme = new Filme(null, null, null, null,
      LocalDate.of(2017, 10, 20),
      null);
  LocalDate referencia = LocalDate.of(2017, 10, 22);
  // processamento
  boolean emExibicao = filme.emExibicao(referencia);
  // verificação
  Assert.assertFalse(emExibicao);
}
```

```java
public boolean emExibicao(LocalDate referencia) {
  if (getInicioExibicao() == null || getFimExibicao() == null) {
    return false;
  }
  LocalDate inicio = getInicioExibicao();
  LocalDate fim = getFimExibicao();
  boolean depoisDoInicio = inicio.isEqual(referencia) || inicio.isBefore(referencia);
  boolean antesDoFim = fim.isEqual(referencia) || fim.isAfter(referencia);
  return depoisDoInicio && antesDoFim;
}
```

Note que é bem provável que chegaríamos na mesma solução sem TDD, a diferença crítica
aqui é no futuro, quando tivermos que trocar uma dessas regras: a suíte de teste vai
ser valiosa na identificação de impactos não considerados ou defeitos de regressão
que por ventura adicionaremos no projeto.

E assim terminamos a versão mocada. [Vamos falar agora sobre DevOps](06_devops.md).
