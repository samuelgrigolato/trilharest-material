# Usando Swagger para documentar nossa API

Uma informação valiosíssima para nossa equipe de front-end é a documentação da API.
Como eles vão saber as URL e os tipos de dados que devem passar/receber ao chamar nossos
endpoints? Seria entediante gerar tudo isso na mão e manter atualizado, por isso
utilizaremos uma ferramenta chamada `Swagger`, através de uma biblioteca que faz a
integração dela com o Spring Boot chamada `Springfox`.

A integração é bem simples, adicionaremos primeiro duas dependências no nosso
`build.gradle`:

```groovy
compile "io.springfox:springfox-swagger2:2.7.0"
compile group: 'io.springfox', name: 'springfox-swagger-ui', version: '2.7.0'
```

Depois atualizaremos os descritores do projeto para a IDE:

```
$ gradle idea
```

E habilitaremos o `Swagger` na aplicação, anotando a classe `FilmesApplication` com
a anotação `@EnableSwagger2`:

```java
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@EnableAutoConfiguration
@ComponentScan
@EnableSwagger2
public class FilmesApplication {
```

Pronto, basta executarmos a aplicação e já teremos uma documentação bem rudimentar
disponível nessa URL: http://localhost:8080/swagger-ui.

Essa documentação já ajuda muito, mas em alguns casos pode ser útil adicionar uma
descrição melhor para ajudar os consumidores da nossa API. Para isso o Swagger fornece
uma grande quantidade de anotações, cuja lista completa pode ser vista aqui:
https://github.com/swagger-api/swagger-core/wiki/Annotations-1.5.X. Vamos usar algumas
delas:

```java
import io.swagger.annotations.Api;

@RestController
@RequestMapping("/filmes")
@Api("API para acesso e manipulação dos filmes em cartaz")
public class FilmesAPI {
```

```java
import io.swagger.annotations.ApiOperation;
[...]
public class FilmesAPI {
[...]

    @GetMapping("/em-exibicao")
    @ApiOperation(value="Buscar página de filmes em exibição",
                notes="Permite a busca paginada de filmes em exibição, " +
                        "ou seja, filmes que possuam data de início e término " +
                        "de exibição e cujo período engloba a data atual")
    public Pagina<Filme> get(ParametrosDePaginacao parametrosDePaginacao) {
```

```java
import io.swagger.annotations.ApiParam;
[...]
public class ParametrosDePaginacao {
[...]

    @ApiParam(value="Número da página desejada", defaultValue="1")
    private Integer pagina = 1;

    @ApiParam(value="Tamanho máximo da página",  defaultValue="3")
    private Integer tamanhoDaPagina = 3;
```

```java
import io.swagger.annotations.ApiModelProperty;
[...]
public class Filme {
[...]

    @ApiModelProperty(value="Duração do filme sem trailers")
    private Duration duracao;
```

Uma sugestão antes de partirmos para o próximo tópico: comece com o básico (anote com
`@EnableSwagger2`) e vá incrementando apenas onde necessário, não estipule regras de
obrigatoriedade para por exemplo documentar todos os parâmetros, pois acabará com
documentações desnecessárias e óbvias como "id: Este parâmetro é o identificador do
registro".

Agora discutiremos sobre base de dados, clique [aqui](08_armazenamento.md)
para ir para o próximo tópico.
