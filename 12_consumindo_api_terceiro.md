# Consumindo uma API de terceiro

Dê uma ohada neste site: https://www.themoviedb.org/?language=pt-BR. Não seria muito
interessante se pudéssemos trazer as informações dele para nossa plataforma? Pois é
exatamente isso que iremos fazer neste tópico.

Hoje em dia é muito comum os serviços online mais conhecidos fornecerem uma API de
dados para que desenvolvedores integrem suas próprias aplicações. No caso do The Movie
DB, eles fornecem uma série de endpoints gratuitamente (para fins não comerciais)
para acesso à base de dados de filmes que eles possuem. Veja a documentação para
ter uma ideia das possibilidades: https://developers.themoviedb.org/3.

Aqui no nosso caso usaremos o endpoint de busca de filmes futuros (upcoming), para
periodicamente trazer essas novidades automaticamente para nossa base de dados. O
endpoint é esse aqui:

> https://developers.themoviedb.org/3/movies/get-upcoming

Repare que em todos os casos existe este parâmetro chamado `api_key`. Este parâmetro
é a autenticação, é o que permite que o The Movie DB saiba que você que efetuou essa
chamada. Essa prática é muito comum e permite a definição de limites de uso,
ticketagem (cobrança por requisição), auditoria etc. Toda conta no site pode
solicitar uma chave de API, e se o uso for não comercial a aprovação costuma ser
imediata e automática.

Em posse da sua API Key, podemos testar o endpoint com a seguinte URL (use o Postman
para que a resposta já venha bem formatada e fácil de ler):

> https://api.themoviedb.org/3/movie/upcoming?api_key=INSIRA_O_TOKEN_AQUIlanguage=pt-BR&page=1

Você também pode usar a aba `Try it out` disponível na página da documentação.

```json
{
    "results": [
        {
            "vote_count": 541,
            "id": 440021,
            "video": false,
            "vote_average": 6.4,
            "title": "A Morte Te Dá Parabéns",
            "popularity": 506.307234,
            "poster_path": "/n5Zd5QIrAWOEYW7V7Tm7MiMC2aE.jpg",
            "original_language": "en",
            "original_title": "Happy Death Day",
            "genre_ids": [
                27
            ],
            "backdrop_path": "/eGx5OfOdvM0gkHdmkLe3hcJuEIT.jpg",
            "adult": false,
            "overview": "No longa, a personagem Tree Gelbman (Jessica Rothe), é assassinada por uma pessoa mascarada no dia de seu aniversário. Para descobrir o responsável pelo crime, ela ressuscita várias vezes e se vê presa em um ciclo entre vida e morte até conseguir solucionar seu próprio assassinato. Ela só conseguirá escapar de um destino trágico, quando compreender as reais causas de sua morte.",
            "release_date": "2017-10-12"
        },
[...]
```

Mas como consumir essa API de dentro da nossa aplicação Java? E ainda, como fazer isso
dentro do contexto Spring para que possamos inserir os novos registros de filme?
O que nós precisamos é de uma forma de subir a aplicação sem o container web, e isso
é possível criando uma classe `Application` diferente. Vamos criar uma classe chamada
`BatchApplication` no mesmo pacote onde se encontra a `FilmesApplication`:

```java
package com.opensanca.trilharest.filmes;

import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;

@EnableAutoConfiguration
@ComponentScan
@EnableWebSecurity
@EnableScheduling
public class BatchApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(BatchApplication.class)
            .web(false)
            .run(args);
    }

}
```

Atenção para o `.web(false)`, isso faz com que o container web não suba nesse processo.
O segundo é o `@EnableScheduling`, que nos permite definir métodos em beans que
são executados periodicamente (de uma em uma hora, diariamente, etc). Rode essa nova
aplicação (não se esqueça do parâmetro de profile Spring `-Dspring.profiles.active=dev`).
O `@EnableWebSecurity` não deveria ser necessário mas sem ele a aplicação não sobe,
por conta do grafo de dependências de nossas classes de configuração.

Agora podemos criar um bean que conterá nossa rotina batch. Vamos chamá-lo de
`ImportadorTheMovieDB` e colocá-lo no pacote `com.opensanca.trilharest.filmes.themoviedb`:

```java
package com.opensanca.trilharest.filmes.themoviedb;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.boot.autoconfigure.condition.ConditionalOnNotWebApplication;


@Component
@ConditionalOnNotWebApplication
public class ImportadorTheMovieDB {

    private static final Logger LOGGER = LoggerFactory.getLogger(ImportadorTheMovieDB.class);

    @Scheduled(fixedDelay = 15 * 1000)
    public void executar() {
        LOGGER.info("Executando importação de filmes do TheMovieDB...");
    }

}
```

O parâmetro `fixedDelay` por padrão é em milissegundos. Esse delay é contado *entre*
as execuções, em contrapartida o parâmetro `fixedRate` conta entre os *inícios* de
execução, o que pode acarretar em execução paralela e normalmente não é o que
precisamos.

Agora que já temos um ponto no código que executa periodicamente, precisamos implementar
a chamada a API do The Movie DB. Usaremos para isso uma classe utilitária do Spring
chamada `RestTemplate`:

```java
package com.opensanca.trilharest.filmes.themoviedb;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.util.List;


@Component
public class ImportadorTheMovieDB {

    private static final Logger LOGGER = LoggerFactory.getLogger(ImportadorTheMovieDB.class);

    @Value("${themoviedb.api_key}")
    private String apiKey;

    @Scheduled(fixedDelay = 15 * 1000)
    public void executar() {
        LOGGER.info("Executando importação de filmes do TheMovieDB...");

        RestTemplate api = new RestTemplate();

        String url = "https://api.themoviedb.org/3/movie/upcoming?language=pt-BR&page=1&api_key=" + apiKey;

        FilmesTheMovieDB filmes = api.getForObject(url, FilmesTheMovieDB.class);
        System.out.println(filmes.getResults().size());
    }

}
```

```java
package com.opensanca.trilharest.filmes.themoviedb;

import java.util.List;

public class FilmesTheMovieDB {

    private List<FilmeTheMovieDB> results;

    // getters e setters
}
```

```java
package com.opensanca.trilharest.filmes.themoviedb;

public class FilmeTheMovieDB {
}
```

Note que criamos uma propriedade para conter nossa API Key. Essa propriedade pode ser
fornecida via `application.properties` ou (neste caso é preferível) via argumento
de linha de comando (assim como fizemos com o `-Dspring.profiles.active=dev` podemos
fazer também `-Dthemoviedb.api_key=XXXXXXXXX`).

Agora podemos ir adicionando as propriedades que nos interessam na classe
`FilmesTheMovieDB`, para por fim fazer o mapeamento em entidades do tipo `Filme`
e persistir na nossa base de dados:

```java
package com.opensanca.trilharest.filmes.themoviedb;

import com.opensanca.trilharest.filmes.filmes.Filme;
import com.opensanca.trilharest.filmes.filmes.FilmesRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnNotWebApplication;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;


@Component
@ConditionalOnNotWebApplication
public class ImportadorTheMovieDB {

    private static final Logger LOGGER = LoggerFactory.getLogger(ImportadorTheMovieDB.class);

    @Value("${themoviedb.api_key}")
    private String apiKey;

    @Autowired
    private FilmesRepository filmesRepository;

    @Scheduled(fixedDelay = 15 * 1000)
    public void executar() {
        LOGGER.info("Executando importação de filmes do TheMovieDB...");

        RestTemplate api = new RestTemplate();

        String url = "https://api.themoviedb.org/3/movie/upcoming?language=pt-BR&page=1&api_key=" + apiKey;

        FilmesTheMovieDB filmes = api.getForObject(url, FilmesTheMovieDB.class);

        List<Filme> entidades = filmes.getResults().stream()
            .map(x -> {
                Filme entidade = new Filme();
                entidade.setId(UUID.randomUUID());
                entidade.setNome(x.getTitle());
                entidade.setSinopse(x.getOverview());
                return entidade;
            })
            .collect(Collectors.toList());

        filmesRepository.save(entidades);
    }

}
```

```java
package com.opensanca.trilharest.filmes.themoviedb;

public class FilmeTheMovieDB {

    private String title;
    private String overview;

    // getters e setters
}
```

Pronto, agora temos uma rotina que traz os dados do The Movie DB para nosso banco de
dados local. Um primeiro passo para melhorá-la seria evitar que filmes já existentes
na base sejam duplicados, mas até o ponto onde chegamos já é possível ter uma boa
ideia de como desenvolver este tipo de integração usando Spring.
