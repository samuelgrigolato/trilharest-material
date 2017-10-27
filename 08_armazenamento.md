# Substituindo o mecanismo de armazenamento

Até agora temos usado persistência dos dados em memória. Isso é útil para prototipações
rápidas mas nem tanto em ambientes de produção, afinal queremos que nossos dados
persistam a reinicializações de servidor.

Para resolver este problema precisamos substituir a implementação de `FilmesRepository`
atualmente usada pela aplicação, ou seja, trocar a classe `FilmesRepositoryRAM` por
uma classe que conecta a ideia de negócio `FilmesRepository` com um provedor de
armazenamento diferente. Utilizaremos uma base relacional `postgres` para este fim,
em conjunto com a biblioteca *Spring Data*.

Vamos começar adicionando o *starter-pom Spring Data JPA* no projeto. Isso é feito
no arquivo `build.gradle`, na seção de dependências:

```groovy
dependencies {
    [...]
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-web', version: '1.5.7.RELEASE'
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-data-jpa', version: '1.5.7.RELEASE'
    [...]
}
```

Depois temos que atualizar os descritores do projeto na IDE:

```
$ gradle idea
```

Agora vamos criar uma outra implementação para a interface `FilmesRepository`, chamada
`FilmesRepositoryJPA`:

```java
import com.opensanca.trilharest.movies.comum.Pagina;
import com.opensanca.trilharest.movies.comum.ParametrosDePaginacao;
import java.time.LocalDate;
import java.util.UUID;
import javax.persistence.EntityManager;
import javax.persistence.Query;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Root;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;

@Repository
public class FilmesRepositoryJPA implements FilmesRepository {

  @Autowired
  private EntityManager entityManager;

  @Override
  public Pagina<Filme> buscarPaginaEmExibicao(ParametrosDePaginacao parametrosDePaginacao,
      LocalDate referencia) {

    CriteriaBuilder cb = this.entityManager.getCriteriaBuilder();

    CriteriaQuery<Filme> criteriaSeletora = cb.createQuery(Filme.class);
    Root<Filme> raizSeletora = criteriaSeletora.from(Filme.class);
    criteriaSeletora.where(construirPredicadoEmExibicao(referencia, cb, raizSeletora));
    Query querySeletora = this.entityManager.createQuery(criteriaSeletora);
    querySeletora.setFirstResult((parametrosDePaginacao.getPagina() - 1) * parametrosDePaginacao.getTamanhoDaPagina());
    querySeletora.setMaxResults(parametrosDePaginacao.getTamanhoDaPagina());

    CriteriaQuery<Long> criteriaContadora = cb.createQuery(Long.class);
    Root<Filme> raizContadora = criteriaContadora.from(Filme.class);
    criteriaContadora.select(cb.count(cb.literal(1)));
    criteriaContadora.where(construirPredicadoEmExibicao(referencia, cb, raizContadora));
    Query queryContadora = this.entityManager.createQuery(criteriaContadora);

    Pagina<Filme> pagina = new Pagina<>();
    pagina.setTotalDeRegistros(((Number) queryContadora.getSingleResult()).intValue());
    pagina.setRegistros(querySeletora.getResultList());
    return pagina;
  }

  @Override
  public Filme buscarPorId(UUID id) {
    return this.entityManager.find(Filme.class, id);
  }

  private Predicate construirPredicadoEmExibicao(LocalDate referencia, CriteriaBuilder cb,
      Root<Filme> raiz) {
    return cb.between(cb.literal(referencia), raiz.get("inicioExibicao"), raiz.get("fimExibicao"));
  }
}
```

Se tentarmos rodar a aplicação neste momento teremos o seguinte erro:

```
Cannot determine embedded database driver class for database type NONE
```

Isso ocorre pois não temos nenhum driver JDBC no classpath, tampouco dissemos para o
Spring Data qual é o `data source` padrão dele. Vamos resolver a questão do data
source colocando os seguintes parâmetros no nosso `application.properties`:

```
spring.datasource.url=jdbc:postgresql://127.0.0.1/filmes
spring.datasource.username=postgres
spring.datasource.password=123456
```

Agora o Spring Data vai tentar carregar o driver do postgres, mas não terá sucesso
pois este não está no classpath. Vamos adicioná-lo no arquivo `build.gradle` na
seção de dependências:

```groovy
compile group: 'org.postgresql', name: 'postgresql', version: '42.1.4'
```

Nota: lembre-se de rodar o comando `gradle idea` sempre que alterar o `build.gradle`.

Por fim temos que criar a base de dados `filmes`, podemos fazer isso via `psql` ou
via `pgAdmin`.

Note que depois disso o Spring não vai deixar a aplicação subir, alegando que existem
duas implementações para a interface `FilmesRepository`:

```
Field filmesRepository in com.opensanca.trilharest.movies.filmes.FilmesAPI required a single bean, but 2 were found:
	- filmesRepositoryJPA: defined in file [/home/samuel/wksp/trilharest/hello1/out/production/hello1/com/opensanca/trilharest/movies/filmes/FilmesRepositoryJPA.class]
	- filmesRepositoryRAM: defined in file [/home/samuel/wksp/trilharest/hello1/out/production/hello1/com/opensanca/trilharest/movies/filmes/FilmesRepositoryRAM.class]
```

Como resolver? Uma opção é remover de vez a classe `FilmesRepositoryRAM`, visto que ela
não nos servirá no futuro (de fato é isso que faremos). Uma outra seria *anotar* uma
das classes com a anotação `@Primary`.

Suba a aplicação, e tente recuperar uma lista de filmes em exibição acessando a URL
`http://localhost:8080/filmes?pagina=1&tamanhoDaPagina=3`. Não vai dar certo, pois
não anotamos a entidade `Filme` com os metadados que o JPA precisa:

```java
import javax.persistence.Id;

@Entity
public class Filme {

    @Id
    private UUID id;
```

O último erro antes de conseguirmos de fato executar essa consulta é a criação da
tabela no banco de dados! Quem diria! Temos diversas formas de resolver aqui, a
primeira e mais ingênua seria escrever o DDL `create table` na mão e executar também
manualmente dentro do pgAdmin. Essa estratégia funciona, mas começa a dar trabalho
quando temos vários ambientes. Enviar scripts SQL por e-mail e pedir para um DBA
executar são tarefas comuns mas que hoje em dia não fazemos mais.

A segunda forma é deixar o Hibernate criar tudo sozinho. Veja como podemos fazer
isso com uma simples propriedade no `application.properties`:

```
spring.jpa.hibernate.ddl-auto=update
```

A priori isso parece muito bom, mas essa técnica é indicada *apenas* durante a fase
de prototipação e validação de ideias, sendo ruim quando sua aplicação passa por
evoluções e o banco precisa ser ajustado. Por exemplo, como adicionar um campo
obrigatório dessa forma? Ou refatorar uma estrutura polimórfica que usava coluna
de categoria para uma estrutura que usa várias tabelas do tipo 1 para 1? O
`ddl-auto` não está preparado e não tem a intenção de resolver esse tipo de problema.

Observando a estrutura criada pelo `Hibernate` podemos identificar um outro problema,
ele não entendeu o significado das classes de data do Java 8 (repare como temos
que ajustar isso aqui também, assim como tivemos que ajustar com
o Spring MVC, esse é o custo de usar APIs novas, mas é um custo tranquilo considerando
os benefícios), mapeando todas para o tipo `bytea` (o que significa que ele vai
simplesmente serializar em um array de bytes as instâncias, o que é bem longe do
que realmente queremos).

Para resolver isso temos que configurar um pacote de conversores na classe
`FilmesApplication`:

```java
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.convert.threeten.Jsr310JpaConverters;

@ComponentScan
@EntityScan(basePackageClasses = {FilmesApplication.class, Jsr310JpaConverters.class})
public class MoviesApplication {
```

Repare que se tentar executar sem *dropar* a tabela existente, o `ddl-auto` não
vai dar erro nem alterar o tipo da coluna, o que é um problema. Drope a tabela
via psql ou pgAdmin e suba a aplicação novamente.

Note que resolvemos parcialmente o problema para as colunas de data (não queremos
um timestamp mas sim um date) e não resolvemos nada com relação ao tipo `Duration`.
Infelizmente não parece existir uma solução fácil para a questão do `Duration` que
não seja escrever um conversor manualmente, a não ser que estejamos afim de criar
uma dependência direta com o `Hibernate`, neste caso basta adicionar a seguinte
dependência no `build.gradle`:

```groovy
compile group: 'org.hibernate', name: 'hibernate-java8', version: '5.0.12.Final'
```

Lembre-se do `gradle idea`. Remove as configurações que fizemos no passo anterior
referentes aos conversores Jsr310, drope a tabela existente e suba a aplicação
novamente.

Agora sim! Finalmente temos um mapeamento coeso.

```
  duracao bigint,
  fim_exibicao date,
  inicio_exibicao date,
```

Vamos voltar agora à questão do `ddl-auto`. Não queremos continuar usando-o pois
teremos problemas com as próximas iterações do nosso banco de dados. Por isso
adotaremos uma *ferramenta de migração de banco de dados*, que ficará responsável
por *executar scripts de migração* nos ambientes da nossa solução. As duas
mais conhecidas no mundo Java são o [Liquibase](http://www.liquibase.org/)
e o [Flyway](https://flywaydb.org/). Usaremos o Liquibase na trilha, mas os conceitos
são muito similares entre as duas ferramentas.

O Spring Boot já está preparado para usar o Liquibase, basta adicionarmos ele no
projeto colocando uma dependência no `build.gradle`:

```groovy
compile group: 'org.liquibase', name: 'liquibase-core', version: '3.5.3'
```

Lembre-se do `gradle idea`.

Assim que a aplicação sobe, o Liquibase é disparado e verifica se existe um arquivo
no classpath no caminho `db/changelog/db.changelog-master.yaml`. Vamos criar
esse arquivo com uma migração inicial. Note que esse arquivo pode ser criado dentro
do diretório `src/main/java`, mas é boa prática manter arquivos desse tipo no
diretório `src/main/resources`. Vamos criá-lo então:

```
$ mkdir src/main/resources
$ gradle idea
```

E por fim criar o arquivo `db.changelog-master.yaml` no caminho
`src/main/resources/db/changelog/`:

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: samuelgrigolato
      changes:
      - sqlFile:
          dbms: postgresql
          encoding: utf8
          path: sqls/0001-migracao-inicial.sql
          relativeToChangelogFile: true
```

Repare que estamos delegando a migração para um arquivo SQL, vamos criá-lo no
caminho `src/main/resources/db/changelog/sqls/0001-migracao-inicial.sql`:

```sql
create table filme
(
  id uuid not null,
  duracao bigint,
  fim_exibicao date,
  inicio_exibicao date,
  nome character varying(100) not null,
  sinopse text,

  constraint filme_pkey primary key (id)
);
```

Antes de executar a aplicação não podemos nos esquecer de retirar o `ddl-auto`
do arquivo `application.properties` e dropar a tabela existente.

Vamos adicionar registros equivalentes à versão RAM para testar corretamente:

```sql
insert into filme(id, duracao, fim_exibicao, inicio_exibicao, nome, sinopse)
values ('252f29dc-b5ba-437d-b070-c5b7a7ad128b', 153::bigint * 60 * 1000 * 1000 * 1000, '2017-10-20', '2017-10-1', 'Filme 1', 'Sinopse do filme 1'),
       ('75fc717d-f893-4e7b-8cb7-461ee34243c4', null, '2017-11-10', '2017-09-02', 'Filme 2', null),
       ('06a25754-2f4c-4b9a-b1a0-71b14ce99faa', null, null, null, 'Filme 3', null),
       ('29313c21-087a-4a6c-b972-01f5fe445087', 120::bigint * 60 * 1000 * 1000 * 1000, '2017-10-19', '2017-10-02', 'Filme 4', 'Sinopse do filme 4');
```

Vamos agora dar uma olhada na classe `FilmesRepositoryJPA`. Ela está muito verbosa.
Felizmente podemos melhorar isso consideravelmente usando a API de interfaces
de repositório semânticas do Spring Data. Isso é, ao invés de implementar queries
simples, vamos instruir o `Spring Data` a fazer isso por nós.

A primeira coisa que vamos fazer é remover a classe `FilmesRepositoryJPA`. Depois
vamos ajustar nossa interface `FilmesRepository` para usar os tipos semânticos
do Spring Data:

```java
import java.time.LocalDate;
import java.util.UUID;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.CrudRepository;

public interface FilmesRepository extends CrudRepository<Filme, UUID> {

  @Query("select f from Filme f where ?1 between f.inicioExibicao and f.fimExibicao")
  Page<Filme> buscarPaginaEmExibicao(LocalDate referencia, Pageable pageable);

}

```

Note que temos que atualizar também a classe `FilmesAPI` pois mudamos nosso domínio:

```java
@RequestMapping(path="", method=RequestMethod.GET)
@ApiOperation(value="Buscar página de filmes em exibição",
    notes="Permite buscar uma página de registros em exibição. " +
            "Um filme em exibição é um filme que possui ambas " +
            "datas de início/fim e cujo período contempla a data de hoje")
public Page<Filme> get(Pageable parametrosDePaginacao) {
    LocalDate hoje = LocalDate.now();
    return filmesRepository.buscarPaginaEmExibicao(hoje, parametrosDePaginacao);
}

@RequestMapping(path="/{id}", method=RequestMethod.GET)
@ApiOperation("Buscar filme por ID")
public Filme getPorId(@ApiParam("identificador do filme") @PathVariable UUID id) {
    return filmesRepository.findOne(id);
}
```

Uma coisa bem conveniente aconteceu, como incorporamos as classes de domínio `Page`
e `Pageable` do Spring Data, não precisamos mais das classes `Pagina` e
`ParametrosDePaginacao` no nosso próprio domínio!

Use a seguinte URL para testar: `http://localhost:8080/filmes/em-exibicao?page=0&size=2`.

Antes de passar para o próximo passo, vamos dar uma última melhorada no retorno
da busca por ID. Note que se passar um UUID não existente na base, a API retorna
200 OK com conteúdo vazio. Não é bem isso que queremos (queremos um 404), então
vamos tratar isso na classe `FilmesAPI`:

```java
@RequestMapping(path="/{id}", method=RequestMethod.GET)
@ApiOperation("Buscar filme por ID")
public Filme getPorId(@ApiParam("identificador do filme") @PathVariable UUID id) {
    Filme entidade = filmesRepository.findOne(id);
    if (entidade == null) {
        throw new EntidadeNaoEncontradaException();
    }
    return entidade;
}
```

Ficou um pouco melhor, mas o status code retornado está sendo 500 ao invés de 404.
Isso pode confundir a equipe de front-end, então vamos anotar a classe de exceção
para resolver esse ponto também:

```java
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class EntidadeNaoEncontradaException extends RuntimeException {
}
```

E assim acabamos este tópico. [Clique aqui](09_heroku_addons.md) para ir para o próximo
assunto.
