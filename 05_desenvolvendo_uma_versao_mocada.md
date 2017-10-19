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
Para validar isso, vamos tentar implementar nossa API e ver até onde conseguimos chegar:

```
(pendente, paramos aqui no primeiro dia da trilha)
```
