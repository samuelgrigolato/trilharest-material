# Especificando e implementando os endpoints de administração

Agora que já temos os endpoints de filmes em exibição e detalhamento por ID, vamos
passar para uma próxima iteração do nosso sistema:

> Permitir que administradores cadastrem, alterem e removam filmes.

Ou seja, o tal do CRUD de filmes. Vamos começar desenhando os endpoints REST que teremos
que implementar:

```
GET /filmes: deve fornecer dados paginados de todos os filmes com possibilidade
de ordenação.

POST /filmes: permite o cadastro de um novo filme

PUT /filmes/{id}: permite alterar os dados de um filme existente

DELETE /filmes/{id}: permite excluir um filme existente
```

É primordial que apenas administradores consigam executar essas chamadas, mas veremos
como resolver isso apenas no próximo tópico.

Neste momento temos que conversar um pouco sobre as classes de modelo que estamos
usando na camada de API. Repare que estamos usando a classe `Filme` nas duas buscas,
e possivelmente vocês estão imaginando que a *action* de cadastro ficaria parecida
com isso aqui:

```java
@PostMapping
public void cadastrar(Filme filme) {
    filmesRepository.save(filme);
}
```

Desconsiderando o fato de que não estamos validando nada, essa solução parece ótima! No
entanto, vamos pensar no futuro, em um momento em que eventualmente venhamos a adicionar
a seguinte coluna nessa entidade: `quantidadeDeReclamacoes`. Não queremos disponibilizar
essa informação, que provavelmente é confidencial, para nossos clientes, mas é isso mesmo
que vai acabar acontecendo sem ninguém perceber. Pior ainda, ao cadastrar um novo
registro, um usuário mal intencionado pode forjar uma requisição e **passar um valor
qualquer** para este campo. Imaginem só o estrago em um campo do tipo *saldo* de uma
conta corrente.

Por esses motivos não é boa prática usar modelos nem entidades *muito genéricas* como
saída ou parâmetro de actions MVC, a não ser que elas de fato representem exatamente
o conceito de negócio daquele endpoint. Por exemplo, neste caso teríamos que separar o que
é um `Filme` para os usuários e o que é um `Filme` para os administradores.

Como resolver isso? Uma das maneiras mais simples de resolver é criar classes de
transferência de dados, os famigerados `DTOs`, e usá-los como retorno e parâmetro
das actions MVC. Vamos ver como isso fica no nosso caso. Crie primeiro a classe
`FilmeResumidoDTO` dentro do pacote `com.opensanca.trilharest.filmes.filmes`:

```java
import java.time.Duration;
import java.util.UUID;

public class FilmeResumidoDTO {

  private UUID id;
  private String nome;
  private Duration duracao;

  public FilmeResumidoDTO(UUID id, String nome, Duration duracao) {
    this.id = id;
    this.nome = nome;
    this.duracao = duracao;
  }

  // getters foram omitidos
}
```

E a classe `FilmeDetalhadoDTO` no mesmo pacote:

```java
import java.time.Duration;
import java.time.LocalDate;
import java.util.UUID;

public class FilmeDetalhadoDTO {

  private UUID id;
  private String nome;
  private String sinopse;
  private Duration duracao;
  private LocalDate inicioExibicao;
  private LocalDate fimExibicao;

  public FilmeDetalhadoDTO(Filme entidade) {
    this.id = entidade.getId();
    this.nome = entidade.getNome();
    this.sinopse = entidade.getSinopse();
    this.duracao = entidade.getDuracao();
    this.inicioExibicao = entidade.getInicioExibicao();
    this.fimExibicao = entidade.getFimExibicao();
  }

  // getters foram omitidos
}
```

Vamos agora ajustar nossa classe de API:

```java
  @GetMapping("/em-exibicao")
  @ApiOperation(value="Buscar página de filmes em exibição",
                notes="Permite a busca paginada de filmes em exibição, " +
                      "ou seja, filmes que possuam data de início e término " +
                      "de exibição e cujo período engloba a data atual")
  public Page<FilmeResumidoDTO> get(Pageable parametrosDePaginacao) {
    if (parametrosDePaginacao == null) {
      parametrosDePaginacao = new PageRequest(1, 3);
    }
    LocalDate hoje = LocalDate.now();
    return filmesRepository.buscarPaginaEmExibicao(hoje, parametrosDePaginacao);
  }

  @GetMapping("/{id}")
  public FilmeDetalhadoDTO getPorId(@PathVariable UUID id) {
    Filme filme = filmesRepository.findOne(id);
    if (filme == null) {
      throw new EntidadeNaoEncontradaException();
    }
    return new FilmeDetalhadoDTO(filme);
  }
```

Note na mudança das duas assinaturas e na linha de retorno do método `getPorId`.

Por fim vamos ajustar nossa interface de repositório:

```java
  @Query("select new com.opensanca.trilharest.filmes.filmes.FilmeResumidoDTO( " +
         "  f.id, f.nome, f.duracao) " +
         "from Filme f " +
         "where ?1 between f.inicioExibicao and f.fimExibicao")
  Page<FilmeResumidoDTO> buscarPaginaEmExibicao(LocalDate referencia, Pageable pageable);
```

Por que não fazer isso na API também? No caso de buscas de coleção, costumamos aceitar
o déficit de design para ganhar a performance que vem com não carregar as entidades
inteiras mas apenas os atributos que queremos usar de fato no nosso DTO.

Uma outra maneira de resolver este problema é através do uso de `JSON views`. Vamos
primeiro criar uma estrutura de visões JSON na classe `Filme`:

```java
  public static final class Visualizacoes {
    public static class Resumida {}
    public static class Detalhada extends Resumida {}
  }

  @Id
  @JsonView(Visualizacoes.Resumida.class)
  private UUID id;

  @JsonView(Visualizacoes.Resumida.class)
  private String nome;

  @JsonView(Visualizacoes.Detalhada.class)
  private String sinopse;

  @ApiModelProperty(value="Duração do filme sem trailers")
  @JsonView(Visualizacoes.Resumida.class)
  private Duration duracao;

  @JsonView(Visualizacoes.Detalhada.class)
  private LocalDate inicioExibicao;

  @JsonView(Visualizacoes.Detalhada.class)
  private LocalDate fimExibicao;
```

E anotar o retorno dos métodos da API com a anotação `@JsonView`, dizendo qual
visão queremos usar em cada um dos retornos:

```java
  @GetMapping("/em-exibicao")
  @ApiOperation(value="Buscar página de filmes em exibição",
                notes="Permite a busca paginada de filmes em exibição, " +
                      "ou seja, filmes que possuam data de início e término " +
                      "de exibição e cujo período engloba a data atual")
  @JsonView(Filme.Visualizacoes.Resumida.class)
  public Page<Filme> get(Pageable parametrosDePaginacao) {
    if (parametrosDePaginacao == null) {
      parametrosDePaginacao = new PageRequest(1, 3);
    }
    LocalDate hoje = LocalDate.now();
    return filmesRepository.buscarPaginaEmExibicao(hoje, parametrosDePaginacao);
  }

  @GetMapping("/{id}")
  @JsonView(Filme.Visualizacoes.Detalhada.class)
  public Filme getPorId(@PathVariable UUID id) {
    Filme filme = filmesRepository.findOne(id);
    if (filme == null) {
      throw new EntidadeNaoEncontradaException();
    }
    return filme;
  }
```

E não vamos esquecer de voltar atrás com nossa mudança na interface de repositório:

```java
  @Query("select f " +
         "from Filme f " +
         "where ?1 between f.inicioExibicao and f.fimExibicao")
  Page<Filme> buscarPaginaEmExibicao(LocalDate referencia, Pageable pageable);
```

Note que já de cara temos alguns problemas com essa estratégia: não funcionou
corretamente com a classe de domínio `Page` (teríamos que criar uma classe concreta
customizada para configurar as visões) e o retorno da busca por ID não entendeu
como serializar corretamente as classes de data do Java 8. Por esses motivos nós
não utilizaremos essa abordagem, e ficaremos com a abordagem dos DTOs (portanto
volte o código da API e da interface de repositório que fizemos alguns passos
acima).

Vamos agora criar um DTO para receber dados de cadastro/alteração, chamado
`FilmeFormDTO`:

```java
import java.time.Duration;
import java.time.LocalDate;

public class FilmeFormDTO {

  private String nome;
  private String sinopse;
  private Duration duracao;
  private LocalDate inicioExibicao;
  private LocalDate fimExibicao;

  public void preencher(Filme entidade) {
    entidade.setNome(this.nome);
    entidade.setSinopse(this.sinopse);
    entidade.setDuracao(this.duracao);
    entidade.setInicioExibicao(this.inicioExibicao);
    entidade.setFimExibicao(this.fimExibicao);
  }

  public Filme construir() {
    Filme entidade = new Filme();
    entidade.setId(UUID.randomUUID());
    atualizar(entidade);
    return entidade;
  }

  // getters e setters foram omitidos
}
```

Note que já criamos métodos chamados `preencher` e `construir` que serão úteis
no mapeamento do DTO para entidades.

Com esse DTO em mãos vamos mapear as actions do CRUD:

```java
  @GetMapping
  public Page<FilmeResumidoDTO> getTodos(Pageable parametrosDePaginacao) {
    if (parametrosDePaginacao == null) {
      parametrosDePaginacao = new PageRequest(1, 3);
    }
    return filmesRepository.buscarPagina(parametrosDePaginacao);
  }

  @PostMapping
  public UUID cadastrar(@RequestBody FilmeFormDTO dados) {
    Filme entidade = dados.construir();
    filmesRepository.save(entidade);
    return entidade.getId();
  }

  @PutMapping("/{id}")
  public void alterar(@PathVariable UUID id, @RequestBody FilmeFormDTO dados) {
    Filme entidade = filmesRepository.findOne(id);
    dados.preencher(entidade);
    filmesRepository.save(entidade);
  }

  @DeleteMapping("/{id}")
  public void excluir(@PathVariable UUID id) {
    filmesRepository.delete(id);
  }
```

E criar o método `buscarPagina` na interface de repositório:

```java
  @Query("select new com.opensanca.trilharest.filmes.filmes.FilmeResumidoDTO( " +
         "  f.id, f.nome, f.duracao) " +
         "from Filme f ")
  Page<FilmeResumidoDTO> buscarPagina(Pageable pageable);
```

Note que para testar esses endpoints teremos que usar o Postman, já que não é possível
simular requisições para métodos POST, PUT e DELETE apenas usando a barra do navegador.

Teste as seguintes requisições:

```
POST /filmes
{
    "nome": "Novo filme"
}
```

```
GET /filmes/{id gerado na anterior}
```

```
PUT /filmes/{id gerado na primeira}
{
    "nome": "Nome alterado",
    "sinopse": "Sinopse do filme",
    "duracao": null,
    "inicioExibicao": null,
    "fimExibicao": null
}
```

```
DELETE /filmes/{id gerado na primeira}
```

Note que não passamos nem a duração nem as datas no cadastro e na alteração, por quê?
Da forma como nossa aplicação está configurada hoje ela formata as datas e durações de
maneira bem ruim (execute um `GET /filmes/{algum id existente}` e veja como elas são
serializadas). Precisamos resolver isso pois caso contrário o caminho inverso (receber
esses tipos de dado do usuário da API) também não vai funcionar corretamente.

Para resolver isso adicionaremos uma dependência no `build.gradle`:

```groovy
compile group: 'com.fasterxml.jackson.datatype', name: 'jackson-datatype-jsr310', version: '2.9.1'
```

Só isso já melhorou bastante, mas ainda podemos melhorar mais, adicionando uma
propriedade no `application.properties`:

```
spring.jackson.serialization.write_dates_as_timestamps=false
```

Isso fará com que a saída fique parecida com:

```json
{
    "id": "5a63b561-2097-4f29-acc4-9327fdcede84",
    "nome": "Nome alterado",
    "sinopse": "Sinopse do filme",
    "duracao": "PT2H33M",
    "inicioExibicao": "2017-10-01",
    "fimExibicao": "2018-10-20"
}
```

Note que podemos usar a mesma formatação para a entrada de informações (ou seja,
as actions que respondem por POST e PUT). Observe também que este formato de
duração é o formato ISO para durações, suportado nativamente por várias bibliotecas
JavaScript dentre elas a `moment.js` (ex: `moment.duration("PT2H")`).

O próximo passo agora é falarmos de validação. Note que se passarmos `null` para
o campo `nome` irá estourar um erro no banco de dados, e não queremos isso. Também
não queremos permitir que o usuário informe uma data de fim de exibição menor que
a data de início de exibição, e por fim não queremos permitir que o usuário cadastre
múltimos filmes com o mesmo nome. Repare que cada uma dessas validações tem uma
natureza distinta (uma depende de um único dado, outra depende de muitas informações
do mesmo registro e a última depende de informações do banco de dados). Por isso,
cada uma delas vai exigir uma estratégia diferente para ser implementada.

Vamos começar pela validação de obrigatoriedade. Podemos usar a suíte de anotações
da especificação `JSR 303` (Bean Validation) para isso, visto que o Spring MVC já
está integrado com essa suite. Vamos começar dizendo ao Spring MVC que ele deve
não apenas processar o corpo da requisição, mas validá-lo usando JSR 303, através
da anotação `@Valid` no parâmetro que recebe os dados da requisição. Além disso,
adicionamos um parâmetro imediatamente após do parâmetro anotado com `@Valid`
do tipo `BindingResults`. É nesse objeto que o Spring MVC disponibiliza o resultado
das validações e conversões aplicadas:

```java
[...]
import javax.validation.Valid;
import org.springframework.validation.BindingResult;
[...]
public class FilmesAPI {
[...]

  @PostMapping
  public UUID cadastrar(@Valid FilmeFormDTO dados, BindingResult results) {
    Filme entidade = dados.construir();
    filmesRepository.save(entidade);
    return entidade.getId();
  }

  @PutMapping("/{id}")
  public void alterar(@PathVariable UUID id, @Valid FilmeFormDTO dados, BindingResult results) {
    Filme entidade = filmesRepository.findOne(id);
    dados.preencher(entidade);
    filmesRepository.save(entidade);
  }
```

Mas de nada adianta a validação se não fizermos nada com o resultado. Vamos ajustar o
corpo dos métodos para disparar uma exceção customizada caso um erro ou mais erros
tenham surgido. Primeiro crie uma `BindingException` no pacote
`com.opensanca.trilharest.filmes.comum`:

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
public class BindingException extends RuntimeException {

  private BindingResult results;

  public BindingException(BindingResult results) {
    this.results = results;
  }

  public BindingResult getResults() {
    return results;
  }
}
```

E use-a nas actions de cadastro e alteração:

```java
  @PostMapping
  public UUID cadastrar(@Valid @RequestBody FilmeFormDTO dados, BindingResult results) {
    if (results.hasErrors()) {
      throw new BindingException(results);
    }
    Filme entidade = dados.construir();
    filmesRepository.save(entidade);
    return entidade.getId();
  }

  @PutMapping("/{id}")
  public void alterar(@PathVariable UUID id, @Valid @RequestBody FilmeFormDTO dados, BindingResult results) {
    if (results.hasErrors()) {
      throw new BindingException(results);
    }
    Filme entidade = filmesRepository.findOne(id);
    dados.preencher(entidade);
    filmesRepository.save(entidade);
  }
```

Por fim temos que adicionar anotações JSR 303 na classe `FilmeFormDTO`:

```java
import org.hibernate.validator.constraints.NotBlank;
[...]

public class FilmeFormDTO {

  @NotBlank
  private String nome;
  private String sinopse;
```

Tente agora cadastrar um filme sem informar o nome. Verá que a saída vai mencionar
uma `BindingException`, mas sem os dados detalhados do que ocorreu, o que não nos
ajuda muito. Para resolver esse tipo de problema podemos criar `handlers` específicos
para algumas exceções, modificando a forma como a resposta da requisição é montada.

Para isso vamos criar um método `tratarBindingException` na classe `FilmesAPI`:

```java
  @ExceptionHandler(BindingException.class)
  public BindingExceptionDetalhadoDTO tratarBindingException(BindingException ex) {
    return new BindingExceptionDetalhadoDTO(ex.getResults());
  }
```

E crie também a classe `BindingExceptionDetalhadoDTO` no pacote
`com.opensanca.trilharest.filmes.comum`:

```java
import java.util.List;
import org.springframework.validation.BindingResult;
import org.springframework.validation.ObjectError;

public class BindingExceptionDetalhadoDTO {

  private List<ObjectError> erros;

  public BindingExceptionDetalhadoDTO(BindingResult results) {
    this.erros = results.getAllErrors();
  }

  public List<ObjectError> getErros() {
    return erros;
  }
}
```

Note que o método `tratarBindingException` é genérico, não tem por que ser duplicado
em futuros controllers que venhamos a criar. Para esse tipo de coisa o Spring MVC
fornece a anotação `@ControllerAdvice`, que deve ser usada em classes cujos métodos
serão aplicados a **todos** os outros controladores do projeto. Crie uma classe
chamada `BindingExceptionControllerAdvice` no pacote comum, movendo o método
`tratarBindingException` para dentro dele:

```java
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestController;

@ControllerAdvice
@RestController
public class BindingExceptionControllerAdvice {

  @ExceptionHandler(BindingException.class)
  public BindingExceptionDetalhadoDTO tratarBindingException(BindingException ex) {
    return new BindingExceptionDetalhadoDTO(ex.getResults());
  }

}
```

E assim resolvemos nossa primeira categoria de erro! E como resolver o segundo tipo?
Lembrando, este segundo tipo é o tipo onde precisamos de mais de uma informação
dentro do mesmo formulário. Esse ainda é resolvível dentro do JSR 303, usando restrições
de nível de classe. Primeiro crie a anotação `FimExibicaoAposInicio` no pacote
`com.opensanca.trilharest.filmes.filmes`:

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import javax.validation.Constraint;

@Constraint(validatedBy = FimExibicaoAposInicioValidator.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface FimExibicaoAposInicio {
  String message() default "Fim de exibição deve ser uma data após o início da exibição";
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};
}
```

Depois crie a classe de validação `FimExibicaoAposInicioValidator` no mesmo pacote:

```java
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class FimExibicaoAposInicioValidator implements ConstraintValidator<FimExibicaoAposInicio, FilmeFormDTO> {

  @Override
  public void initialize(FimExibicaoAposInicio constraintAnnotation) {
  }

  @Override
  public boolean isValid(FilmeFormDTO value, ConstraintValidatorContext context) {
    if (value == null) {
      return true;
    }
    if (value.getInicioExibicao() == null || value.getFimExibicao() == null) {
      return true;
    }
    return !value.getFimExibicao().isBefore(value.getInicioExibicao());
  }
}
```

E por fim anote a classe `FilmeFormDTO` com a anotação de restrição:

```java
@FimExibicaoAposInicio
public class FilmeFormDTO {
```

E assim validamos o segundo tipo de erro. Agora vamos ao último! Como fazer para
validar se existe um outro filme com o mesmo nome? Note que para fazer essa validação
nós precisamos de outros dados do banco. Uma boa notícia: no `Spring MVC` você
*pode* injetar dependências em contraint validators, ou seja, poderíamos criar uma
restrição de classe como fizemos com a `@FimExibicaoAposInicio`, injetar o repositório
de filmes nela e validar nossa regra. O único problema com essa abordagem é que o
nosso DTO de filmes **não** tem a informação do ID no caso da alteração, logo como
saber se o registro com mesmo nome não é o próprio registro que estamos alterando?
Para esse tipo de situação podemos augmentar o objeto `BindingResults` dentro da
action do controller, assim:

```java
  @PostMapping
  public UUID cadastrar(@Valid @RequestBody FilmeFormDTO dados, BindingResult results) {
    validarNomeDuplicado(dados, null, results);
    if (results.hasErrors()) {
      throw new BindingException(results);
    }
    Filme entidade = dados.construir();
    filmesRepository.save(entidade);
    return entidade.getId();
  }

  @PutMapping("/{id}")
  public void alterar(@PathVariable UUID id, @Valid @RequestBody FilmeFormDTO dados, BindingResult results) {
    validarNomeDuplicado(dados, id, results);
    if (results.hasErrors()) {
      throw new BindingException(results);
    }
    Filme entidade = filmesRepository.findOne(id);
    dados.preencher(entidade);
    filmesRepository.save(entidade);
  }

  private void validarNomeDuplicado(FilmeFormDTO dados, UUID id, BindingResult results) {
    if (dados != null && dados.getNome() != null) {
      Filme existente = this.filmesRepository.findByNome(dados.getNome());
      if (existente != null && !existente.getId().equals(id)) {
        results.reject("Já existe um filme com este nome!");
      }
    }
  }
```

Criando a nova dependência na interface de repositório:

```java
public interface FilmesRepository extends CrudRepository<Filme, UUID> {
[...]

  Filme findByNome(String nome);
}
```

Antes de encerrar o tópico vamos discutir um pouco sobre essas validações no controlador.
Elas não deveriam estar aí, certo? O local ideal para fazer essas coisas seria em um
componente na camada de modelo, um *service*, que recebe o DTO e dispara uma exceção
se houver um filme com mesmo nome. O problema é que dessa forma não conseguiríamos
entregar todas as mensagens de erro de uma só vez para nosso frontend, mas sim primeiro
entregaríamos as mensagens básicas e depois as de negócio, resultando em uma má
experiência de usuário. Para validações simples como as que vimos aqui a solução
apresentada é suficiente, mas para casos mais complicados valeria muito a pena criar
uma classe de serviço para abstrair a lógica de negócio e poder ser reaproveitada
caso mais partes do sistema tenham que cadastrar um filme.

Outro ponto que podemos melhorar é retirar o texto das mensagens de validação de dentro
do código Java. Podemos ao invés disso movê-las para um arquivo chamado
`messages.properties`, a ser criado no diretório `src/main/resources`:

```
com.opensanca.trilharest.filmes.filmes.FimExibicaoAposInicio.message=Fim de exibição deve ser uma data após o início da exibição 2
filme.duplicado=Já existe um filme com este nome!
```

Ajustando a classe da anotação `FimExibicaoAposInicio`:

```java
public @interface FimExibicaoAposInicio {
  String message() default "{com.opensanca.trilharest.filmes.filmes.FimExibicaoAposInicio.message}";
```

E a classe `FilmesAPI`:

```java
import org.springframework.context.MessageSource;
[...]
public class FilmesAPI {
[...]
  @Autowired
  private MessageSource messageSource;
[...]
  private void validarNomeDuplicado(FilmeFormDTO dados, UUID id, BindingResult results) {
    if (dados != null && dados.getNome() != null) {
      Filme existente = this.filmesRepository.findByNome(dados.getNome());
      if (existente != null && !existente.getId().equals(id)) {
        results.reject("filme.duplicado",
            messageSource.getMessage("filme.duplicado", null, null));
      }
    }
  }
```

Por padrão o validador JSR 303 criado pelo Spring *não* vai enxergar as mensagens
no bundle *messages.properties*. Para resolver isso temos que personalizar o bean
`localValidatorFactoryBean`, adicionando o seguinte método na classe
`FilmesApplication`:

```java
[...]
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
[...]
  @Bean
  public LocalValidatorFactoryBean localValidatorFactoryBean(@Autowired MessageSource messageSource) {
    LocalValidatorFactoryBean validatorFactoryBean = new LocalValidatorFactoryBean();
    validatorFactoryBean.setValidationMessageSource(messageSource);
    return validatorFactoryBean;
  }
```

E assim finalizamos o CRUD de filmes!
