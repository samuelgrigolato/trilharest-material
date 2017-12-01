# Autenticação/autorização

Agora que os endpoints de administração estão prontos, podemos nos questionar: não bastaria
para qualquer um instalar o Postman para conseguir *forjar* requisições de administração,
como solicitar a criação ou cancelamento de um filme em cartaz? Sim! Hoje nossa aplicação
está completamente vulnerável a esse tipo de ataque, e é por isso que temos que ficar atentos
sempre que estivermos desenvolvendo uma API de dados:

> Garanta primeiro a segurança da API, e NUNCA leve em conta artifícios de frontend (como
esconder um campo ou tratar alguma condição em JavaScript) nesse processo. Por exemplo, não é
por que o frontend some com o botão *Cancelar* se o usuário não for administrador que o endpoint
DELETE /filmes/:id não deve verificar novamente. Como vimos, basta um `curl` ou um aplicativo
como o Postman para que qualquer um passe por trás de todo seu frontend.

A segurança de qualquer aplicação normalmente envolve dois conceitos, *autenticação* e
*autorização*.

* **Autenticação**: é o processo de *confirmar* que um usuário diz ser quem é. Isso pode ser feito
com um formulário de login/senha, com uma autenticação delegada para terceiros (Facebook, conta
Google etc), integração com serviços de diretório (AD, LDAP) etc. Essa parte não diz o que cada
um pode fazer, apenas garante que uma *sessão de uso* está relacionada com um usuário específico.

* **Autorização**: é o processo de descobrir *o que um usuário pode fazer*. Essa parte normalmente
não se preocupa com credenciais, apenas assume que o módulo de autenticação fez seu trabalho e
*confia* que o usuário é quem diz ser, e à partir dessa *identidade* busca os grupos, papéis,
perfis e permissões relacionadas a ele.

Em conjunto, autenticação e autorização garantem a segurança de uma API de dados.

Vamos começar a implementação então. Precisamos de um endpoint para a parte de autenticação, que
recebe o usuário e a senha, valida essas informações, e de alguma forma retorna um *passaporte*
para futuras requisições. Esse endpoint será o `POST /login`, e o *passaporte* será primeiramente
um *cookie*, mas depois mudaremos isso para um header customizado e por fim utilizaremos o header
padrão *Authorization* em conjunto com um *JSON Web Token* (JWT).

Por quê (não) usar cookies? Um `cookie` nada mais é do que um valor dentro de um armazenamento
especial do navegador, que pode ser manipulado de maneira transparente pelo header também
especial chamado `Cookies`, que pode ser usado tanto em requisições quanto retornado em respostas
do servidor. Basicamente o navegador:

* Coleta todos os cookies relacionados ao domínio de todas as requisições (páginas, imagens,
*requisições ajax*) e embute eles no header `Cookies` da requisição;
* Adiciona todos os cookies que o servidor manda na resposta de requisições nesse armazenamento
especial.

Esse mecanismo funciona *muito* bem para a definição de *sessões web*, veja:

1. O usuário abre seu site pela primeira vez. O servidor não encontra nenhum cookie, gera um
valor pseudo-aleatório e o retorna como um cookie chamado `SessaoDaMinhaApp`;
2. O usuário continua navegando na aplicação, e a cada requisição o navegador embute o valor
do cookie `SessaoDaMinhaApp` no header `Cookies`, e assim o servidor consegue saber que continua
sendo a mesma pessoa.
3. Eventualmente se o usuário efetua login na aplicação, o servidor apenas faz um mapeamento
do valor pseudo-randômico (chamado de identificador da sessão) com o e-mail ou nome de usuário,
passsando a saber assim não só que se trata da mesma pessoa mas quem essa pessoa é, sem precisar
ficar pedindo o usuário e senha sempre.

> Por que motivo então cookies são tão mal vistos? Justamente por serem transparentes, o simples
fato de adicionar uma imagem ou fazer qualquer requisição ajax para um site de terceiro, por
exemplo um site de publicidade, permite que um serviço rastreie todo seu comportamento na web,
informação valiosa e que de certa forma denigre a privacidade individual. Muitas vezes isso é
feito sem consentimento do usuário, o que gerou a má fama que os cookies possuem hoje.

Vamos então implementar nossa autenticação baseada em cookies! Para isso, vamos começar criando
o endpoint `POST /login`, que recebe um objeto JSON com o usuário e a senha desejados e retorna
`200 OK` se a autenticação deu certo (após adicionar o usuário **na sessão**, o que vai acarretar
no retorno do cookie identificador de sessão como poderemos ver), ou `401 Unauthorized` se as
credenciais estiverem inválidas.

Crie uma classe chamada `LoginAPI` no pacote `com.opensanca.trilharest.filmes.seguranca`
com a seguinte definição:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/login")
public class LoginAPI {

    @PostMapping
    public void login(@RequestBody CredenciaisDTO credenciais) {
    }

}
```

Crie também uma classe chamada `CredenciaisDTO` no mesmo pacote:

```java
public class CredenciaisDTO {

    private String usuario;
    private String senha;

    public String getUsuario() {
        return usuario;
    }

    public void setUsuario(String usuario) {
        this.usuario = usuario;
    }

    public String getSenha() {
        return senha;
    }

    public void setSenha(String senha) {
        this.senha = senha;
    }
}
```

Vamos agora criar uma tabela de usuários no banco de dados, começando pela classe de entidade
`Usuario`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import javax.persistence.Entity;
import javax.persistence.Id;
import java.util.UUID;

@Entity
public class Usuario {

    @Id
    private UUID id;
    private String usuario;
    private String senha;

    // getters e setters omitidos
}

```

Precisamos também de uma migração liquibase para adicionar a tabela nas bases relacionais. O
primeiro arquivo a ser modificado é o `db.changelog-master`, adicionando um novo changeset:

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
  - changeSet:
      id: 2
      author: samuelgrigolato
      changes:
      - sqlFile:
          dbms: postgresql
          encoding: utf8
          path: sqls/0002-adicao-da-tabela-de-usuarios.sql
          relativeToChangelogFile: true
```

E por fim o arquivo `0002-adicao-da-tabela-de-usuarios.sql` no diretório
`src/main/resources/db/changelog/sqls`:

```sql
create table usuario (
    id uuid not null,
    usuario character varying(100) not null,
    senha char(64) not null,

    constraint pk_usuario primary key (id)
);
```

Repare que estamos usando o tipo `char(64)` para a senha. Qual o motivo? Não
é muito interessante armazenar senhas no que chamamos de *texto plano*, ou seja,
da forma como são informadas pelo usuário. É uma ótima prática usar *funções
de espalhamento* (ou funções de *hash*) unidirecionais, ou seja, dado um resultado
de execução dessa função não é possível descobrir qual o valor que o gerou.

Funções de hash normalmente possuem uma saída de tamanho único para qualquer
tamanho de entrada, e no caso do SHA256 são 256 bits que podem ser codificados
em 64 caracteres hexadecimais (256 bits = 32 bytes, cada byte pode ser
representado por dois caracteres hexadecimais, visto que cada caractere
hexadecimal pode representar 16 valores distintos e 16*16=256, que é a
quantidade de valores possíveis de um byte).

Alguns exemplos de função de *hash*: MD5, SHA1, SHA256 etc. Hoje em dia não se
considera mais o MD5 e o SHA1 como funções seguras para fins de autenticação,
então utilizaremos a função SHA256. Para inserir usuários de exemplo podemos
usar serviços online de geração de hash, como por exemplo o
`http://www.xorbin.com/tools/sha256-hash-calculator`. Não usaremos *apenas*
a senha na geração do hash, utilizaremos um **prefixo** em todas as senhas,
chamado de *salt*, para aumentar ainda mais a segurança da nossa aplicação.
Esse salt costuma ficar no arquivo `application.properties` ou até mesmo ser
carregado de variável de ambiente, para que possa ser diferente em
desenvolvimento, teste e produção.

Insira alguns usuários na tabela, usando o *salt* `a5KeT1` como prefixo de
senha. Por exemplo, para a senha `123456`, o hash SHA256 deve ser computado
à partir da entrada `a5KeT1123456`, que dá o seguinte resultado:

> 80054d123d90990e012bad576ac9e23f670ddcc8f1ab4698e00b9c1e3ee7860a

```sql
insert into usuario (id, usuario, senha) 
values ('1c729117-e65d-4a36-b171-5f4a73586b63', 'adm1', '80054d123d90990e012bad576ac9e23f670ddcc8f1ab4698e00b9c1e3ee7860a'),
       ('f8fcc35b-5515-4e06-9c47-7332c9494918', 'adm2', '80054d123d90990e012bad576ac9e23f670ddcc8f1ab4698e00b9c1e3ee7860a');
```

Agora que temos nossos usuários na base vamos implementar um repositório
e um método para buscar um registro de usuário pelo seu login. Crie uma
interface chamada `UsuariosRepository` no pacote `com.opensanca.trilharest.filmes.seguranca`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.data.repository.CrudRepository;

import java.util.Optional;
import java.util.UUID;

public interface UsuariosRepository extends CrudRepository<Usuario, UUID> {

    Optional<Usuario> findByUsuario(String usuario);

}
```

Mas e o texto da consulta? Não precisamos dele, pois usamos um nome de método que o Spring
Data consegue entender e derivar a consulta para nós. Veja mais detalhes aqui:
https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation.

Por fim podemos implementar a autenticação na classe `LoginAPI`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import com.opensanca.trilharest.filmes.comum.BindingException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;
import javax.xml.bind.DatatypeConverter;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Arrays;


@RestController
@RequestMapping("/login")
public class LoginAPI {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Value("${autenticacao.salt}")
    private String salt;

    @PostMapping
    public void login(@Valid @RequestBody CredenciaisDTO credenciais, BindingResult results) {
        if (results.hasErrors()) {
            throw new BindingException(results);
        }
        Usuario usuario = this.usuariosRepository.findByUsuario(credenciais.getUsuario())
                .orElseThrow(CredencialInvalidaException::new);
        if (!isSenhaCorreta(credenciais.getSenha(), usuario.getSenha())) {
            throw new CredencialInvalidaException();
        }
        // falta algo aqui!!!
    }

    private boolean isSenhaCorreta(String senha, String espalhamento) {
        MessageDigest digest = null;
        try {
            digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest((salt + senha).getBytes(StandardCharsets.UTF_8));
            byte[] hashDoEspalhamento = DatatypeConverter.parseHexBinary(espalhamento);
            return Arrays.equals(hash, hashDoEspalhamento);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("SHA-256 não suportado pelo servidor");
        }
    }

}
```

Antes de testar precisamos adicionar a seguinte propriedade no arquivo `application.properties`:

```
autenticacao.salt=a5KeT1
```

Vamos testar nossa implementação, emitindo as seguintes requisições usando Postman:

```
POST /login
{ usuario: 'naoexistente': senha: '123456' }

POST /login
{ usuario: 'adm1': senha: '123456789' }

POST /login
{ usuario: 'adm1': senha: '123456' }
```

Note que apenas no último caso o retorno será `200 OK`,  nos outros o retorno será `401
Unauthorized`, da forma como desejamos.

Nossa implementação está interessante, mas podemos melhorá-la! O Spring fornece um bean
chamado `PasswordEncripter`, que pode facilitar nossa vida com a parte de verificação de
hash. Para usá-lo temos que adicionar primeiramente o `spring-security` no projeto (isso
é feito no `build.gradle`):

```groovy
dependencies {
    ...
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-security', version: '1.5.7.RELEASE'
    ...
}
```

Atenção: lembre-se de atualizar o projeto na IDE (via `gradle idea` ou equivalente).

Voltando a classe `LoginAPI`, podemos implementá-la de uma maneira mais limpa:

```java
package com.opensanca.trilharest.filmes.seguranca;

import com.opensanca.trilharest.filmes.comum.BindingException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;


@RestController
@RequestMapping("/login")
public class LoginAPI {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping
    public void login(@Valid @RequestBody CredenciaisDTO credenciais, BindingResult results) {
        if (results.hasErrors()) {
            throw new BindingException(results);
        }
        Usuario usuario = this.usuariosRepository.findByUsuario(credenciais.getUsuario())
                .orElseThrow(CredencialInvalidaException::new);
        if (!passwordEncoder.matches(credenciais.getSenha(), usuario.getSenha())) {
            throw new CredencialInvalidaException();
        }
        // falta algo aqui!!!
    }

}
```

Repare que removemos toda a lógica de verificação, encapsulando ela nessa chamada ao método
`matches` do componente `passwordEncoder`. Como o Spring Security *não* provê uma implementação
padrão deste bean, temos que criá-la nós mesmos, o que faremos em uma nova classe chamada
`SegurancaConfiguration` no pacote `com.opensanca.trilharest.filmes.seguranca`. Classes de
configuração são úteis para instanciarmos beans Spring de maneira programática. Também
usaremos ela para desabilitar a parte de autorização do `Spring Security` por enquanto, já
que ainda estamos na solução caseira.

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.crypto.password.StandardPasswordEncoder;


@Configuration
public class SegurancaConfiguration extends WebSecurityConfigurerAdapter {

    @Value("${autenticacao.secret}")
    private String secret;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new StandardPasswordEncoder(secret);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.anonymous();
    }
}
```

Repare que mudamos o nome da propriedade `salt` para secret, isso se dá pelo fato do `salt`
na implementação do `StandardPasswordEncoder` é gerado randomicamente para cada execução, e
este é embutido junto com o hash gerado e por sua vez persistido no banco. Essa mudança deve
ser refletida no `application.properties` também:

```
autenticacao.secret=a5KeT1
```

Antes de conseguirmos fazer funcionar precisamos dar um jeito de atualizar os espalhamentos no
banco, afinal o algoritmo do password encoder do Spring não é nada compatível com nosso simples
hash de `salt + senha`. Existem várias maneiras de fazer isso, a mais fácil delas é usar o bom
e velho debug da sua IDE para inspecionar o resultado dessa chamada aqui:

```java
passwordEncoder.encode("123456")
```

Caso não saiba fazer isso uma opção mais simples é adicionar um
`System.out.println(passwordEncoder.encode("123456"))` logo acima da condicional que analisa
se a senha é válida ou não. Só não se esqueça de tirá-lo de lá :).

Temos um problema! O valor do hash possui 80 caracteres de tamanho, ao invés de 64 (isso se
dá pois esse algoritmo embute mais dados no resultado, como o algoritmo utilizado, quantidade
de iterações etc). Mas sem pânico, usaremos um novo `changeset` liquibase para resolver este
problema. Primeiro o declararemos no `db.changelog-master.yml`:

```yaml
  - changeSet:
      id: 3
      author: samuelgrigolato
      changes:
      - sqlFile:
          dbms: postgresql
          encoding: utf8
          path: sqls/0003-aumenta-tamanho-coluna-senha.sql
          relativeToChangelogFile: true
```

E depois criamos o arquivo SQL referenciado pelo changeset:

```sql
alter table usuario alter column senha type char(80);
```

E por fim podemos ajustar os registros existentes (esse script deve ser executado direto,
não faz sentido colocá-lo como um changeset do liquibase!):

```sql
update usuario set senha = 'COLOQUE_AQUI_O_HASH_OBTIDO_MAIS_ACIMA';
```

Neste ponto podemos testar novamente via Postman que o comportamento será igual à solução
manual! Deu muito mais trabalho, então por quê fazer dessa maneira? Dois motivos: esse
algoritmo do Spring é muito mais seguro do que o nosso algoritmo manual, e também pois
estamos desacoplando nossa classe `LoginAPI` dos internos de criptografia de senha, o que
facilita com que outros componentes da nossa aplicação venham a usar a mesma coisa no futuro,
incentivando o reúso.

Repare que nossa autenticação ainda não presta para absolutamente nada. O que acontece se
executarmos um endpoint de administração via Postman? Ele ainda não está obrigando que o
usuário esteja logado para permitir a operação. Lembrando que ainda *não* estamos usando
o Spring Security para isso, podemos continuar com nossa estratégia manual e adicionar um
*interceptador de requisições*. Crie uma classe chamada `LoginHandlerInterceptor` no pacote
de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class LoginHandlerInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println(request.getRequestURI());
        return super.preHandle(request, response, handler);
    }
}
```

Repare que os interceptadores nos permitem investigar (e bloquear, retornando `false` no método
`preHandle`) de maneira transparente todas as requisições destinadas aos endpoints MVC.

Note que precisamos *registrar* nossos interceptadores em uma outra classe de configuração, que
deve estender `WebMvcConfigurerAdapter`. Criaremos ela com o nome `MvcConfiguration` no pacote
`com.opensanca.trilharest.filmes`:

```java
package com.opensanca.trilharest.filmes;

import com.opensanca.trilharest.filmes.seguranca.LoginHandlerInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class MvcConfiguration extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginHandlerInterceptor());
    }
}
```

Neste ponto execute a aplicação, verá que a URL de todas as requisições são apresentadas no console,
pois passaram pelo método `preHandle` do `LoginHandlerInterceptor`. Mas o que queremos não é
registrar as requisições, mas **bloqueá-las** caso sejam de determinados tipos e o usuário não
esteja autenticado. Vamos fazer assim, verificaremos se o *método* da requisição é diferente de
`GET` e se sim verificaremos se *existe um usuário na sessão* (é aqui que entram os cookies). Veja:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class LoginHandlerInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        if (HttpMethod.resolve(request.getMethod()) == HttpMethod.GET) {
            // todos podem executar buscas de informação
            return true;
        }

        if ("/login".equals(request.getRequestURI())) {
            // o endpoint de auteneticação também deve ser liberado
            return true;
        }

        // falta coisa aqui!
        response.sendError(HttpStatus.UNAUTHORIZED.value());
        return false;
    }
}
```

Mas como verificar se existe um usuário logado? Neste ponto precisaremos do conceito de *sessão*,
e felizmente o `Spring MVC` nos fornece uma maneira muito fácil de lidar com ela, através da
definição de `beans` com o escopo `Session`. Vamos criar uma classe chamada `UsuarioLogadoDTO`, no
pacote de segurança, que representará os dados de autenticação de um usuário da aplicação:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

import java.util.UUID;

@Component
@Scope("session")
public class UsuarioLogadoDTO {
    private UUID id;

    // getters e setters omitidos
}
```

Agora que temos um lugar para armazenar o identificador do usuário logado em uma determinada
sessão, podemos fazer isso, primeiro na autenticação (que dirá *quem* está logado) e depois
na autorização (que dirá *se o usuário logado pode* fazer a operação):

```java
...
import org.springframework.context.annotation.Scope;
...
@RestController
@RequestMapping("/login")
@Scope("prototype")
public class LoginAPI {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UsuarioLogadoDTO usuarioLogadoDTO;

    @PostMapping
    public void login(@Valid @RequestBody CredenciaisDTO credenciais, BindingResult results) {
        if (results.hasErrors()) {
            throw new BindingException(results);
        }
        Usuario usuario = this.usuariosRepository.findByUsuario(credenciais.getUsuario())
                .orElseThrow(CredencialInvalidaException::new);
        if (!passwordEncoder.matches(credenciais.getSenha(), usuario.getSenha())) {
            throw new CredencialInvalidaException();
        }
        this.usuarioLogadoDTO.setId(usuario.getId());
    }

}
```

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.context.ApplicationContext;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
import org.springframework.web.servlet.support.RequestContextUtils;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class LoginHandlerInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        if (HttpMethod.resolve(request.getMethod()) == HttpMethod.GET) {
            // todos podem executar buscas de informação
            return true;
        }

        if ("/login".equals(request.getRequestURI())) {
            // o endpoint de auteneticação também deve ser liberado
            return true;
        }

        ApplicationContext ctx = RequestContextUtils.findWebApplicationContext(request);
        UsuarioLogadoDTO usuarioLogado = ctx.getBean(UsuarioLogadoDTO.class);

        if (usuarioLogado.getId() == null) {
            response.sendError(HttpStatus.UNAUTHORIZED.value());
            return false;
        }

        return true;
    }
}
```

Agora sim, via Postman, veremos que os endpoints de administração de filmes só estarão
disponíveis *após* uma autenticação com sucesso no endpoint `POST /login`. Ainda dentro do Postman
clique no botão `Cookies` que está localizado perto do botão `Send` de uma aba de requisição,
verifique o valor, remova o cookie, dê uma praticada para ficar confortável com essa solução.

Nossa solução funciona, mas não é nada padronizada. A biblioteca `Spring Security` já fornece
tudo isso pra nós, então por que não usar? Para isso, vamos começar removendo a classe
`LoginHandlerInterceptor` e a classe `LoginAPI`. Depois de removê-las (não se esqueça do registro do interceptador na classe `MvcConfiguration`) vamos ajustar a configuração do Spring Security na classe
`SegurancaConfiguration`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.crypto.password.StandardPasswordEncoder;


@Configuration
public class SegurancaConfiguration extends WebSecurityConfigurerAdapter {

    @Value("${autenticacao.secret}")
    private String secret;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new StandardPasswordEncoder(secret);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        // em APIs de dados não usamos CSRF
        http.csrf().disable();

        http.formLogin();

        http.authorizeRequests()
                .antMatchers(HttpMethod.POST, "/**/*").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.PUT, "/**/*").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.DELETE, "/**/*").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.GET, "/**/*").permitAll()
                .anyRequest().denyAll();

    }
}
```

Na chamada ao `formLogin` habilitados o endpoint `/login` do Spring Security,
que será responsável por receber os dados de usuário e senha. Com a cadeia de chamadas
começando por `authorizeRequests` estamos definindo a autorização necessária para cada
tipo de requisição, por exemplo apenas administradores podem executar requisições do tipo
POST enquanto qualquer usuário (mesmo não autenticado) pode executar requisições GET.

Se executarmos a aplicação do jeito que ela está e enviarmos um `POST /login` veremos
que o endpoint nos retorna um HTML default de página de login (junto com mensagens de erro).
Isso não é nem de perto o que queremos, já que nossa API é uma API de dados, então vamos
começar mudando isso para que ele retorne um 401 sempre que a autenticação falha. Podemos
fazer isso através de um `failureHandler` customizado na configuração de `formLogin`:

```java
[...]
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler;
[...]
        http.formLogin()
                .failureHandler(new SimpleUrlAuthenticationFailureHandler());
[...]
```

A classe `SimpleUrlAuthenticationFailureHandler` recebe um parâmetro, que se nulo faz com que
o endpoint não cause um redirect, mas sim retorne o status code 401.

Agora e se enviarmos os parâmetros corretos? Note que esse endpoint não faz parse JSON da
requisição, então temos que enviar os dados de maneira convencional (no Postman isso significa
selecionar a opção `form-data` na aba `Body` ao invés de `raw` > `JSON`). Além disso, os nomes
reconhecidos por padrão são `username` e `password` ao invés de `usuario` e `senha`.

Note que precisamos de uma forma para indicar ao Spring Security como ele deve fazer para
validar credenciais. Isso é papel do `authenticationProvider` que por sua vez faz parte
do `authenticationManager`, e existe um método de configuração na superclassse da nossa
`SegurancaConfiguration` especificamente para construí-lo:

```java
@Configuration
public class SegurancaConfiguration extends WebSecurityConfigurerAdapter {
    
    [...]

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        auth.inMemoryAuthentication()
                .withUser("adm1").password("123456memoria").roles("ADMIN")
                .and()
                .withUser("user2").password("123456memoria").roles();

    }
}
```

Note que obviamente essa autenticação não serve para muitos casos, pois estamos travando os
usuários no código fonte da aplicação. Ao invés de usar a `inMemoryAuthentication` podemos
usar outras opções como a `jdbcAuthentication`:

```java
[...]
import javax.sql.DataSource;
[...]
@Configuration
public class SegurancaConfiguration extends WebSecurityConfigurerAdapter {
[...]
    @Autowired
    private DataSource dataSource;
[...]

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        auth.jdbcAuthentication()
                .dataSource(dataSource)
                .usersByUsernameQuery("select usuario as username, senha as password, true as enabled from usuario where usuario = ?")
                .authoritiesByUsernameQuery("select ? as username, 'ADMIN' as authority")
                .passwordEncoder(passwordEncoder());

    }

[...]
```

Note que estamos usando uma query fixa para carregar os papéis, em uma situação real teríamos
uma tabela onde carregaríamos essa informação, por exemplo: `select usuario as username,
papel as authority from papel_usuario where usuario = ?`. Como não temos essa tabela no nosso
projeto, estamos considerando que todos os usuários logados são administradores (ouch).

Mas e se precisarmos de ainda mais flexibilidade? Neste caso podemos implementar nosso próprio
`authenticationProvider`, com um bean gerenciado e capaz de usar todos os outros
componentes disponíveis. Vamos começar criando uma classe chamada `CustomAuthenticationProvider`
dentro do pacote de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) authentication;
        String nomeDeUsuario = token.getName();
        String senha = token.getCredentials().toString();

        Usuario usuario = usuariosRepository.findByUsuario(nomeDeUsuario)
                .orElseThrow(() -> new BadCredentialsException("Credenciais inválidas"));

        if (!passwordEncoder.matches(senha, usuario.getSenha())) {
            throw new BadCredentialsException("Credenciais inválidas");
        }

        return new UsernamePasswordAuthenticationToken(nomeDeUsuario, senha,
                Arrays.asList(new SimpleGrantedAuthority("ADMIN")));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.isAssignableFrom(UsernamePasswordAuthenticationToken.class);
    }

}
```

Para finalizar temos que substituir a configuração de `jdbcAuthentication` por uma chamada
ao método de configuração `authenticationProvider`:

```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(customAuthenticationProvider);
    }
```

Estamos quase lá! Temos mais dois cenários a tratar: o primeiro é o que fazer no sucesso de
autenticação (hoje o Spring redireciona para a raíz da aplicação, o que não é desejado em
uma API de dados) e o segundo é o redirecionamento automático que está ocorrendo quando
tentamos acessar uma URL sem estar autenticado. Com relação ao primeiro, temos que definir
um novo `authenticationSuccessHandler` com o comportamento desejado. Para isso crie uma classe
chamada `CustomAuthenticationSuccessHandler` no pacote de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.setStatus(HttpStatus.NO_CONTENT.value());
    }
}
```

E registre-a na classe `SegurancaConfiguration`:

```java
[...]
        http.formLogin()
                .failureHandler(new SimpleUrlAuthenticationFailureHandler())
                .successHandler(new CustomAuthenticationSuccessHandler());
[...]
```

E para resolver o segundo caso, precisamos de um `authenticationEntryPoint` customizado. Crie
uma classe chamada `CustomAuthenticationEntryPoint` no pacote de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.http.HttpStatus;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
    }
}
```

E registre-a no `SegurancaConfiguration`:

```java
[...]
@Configuration
public class SegurancaConfiguration extends WebSecurityConfigurerAdapter {
[...]
    @Autowired
    private CustomAuthenticationEntryPoint customAuthenticationEntryPoint;
[...]
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        // em APIs de dados não usamos CSRF
        http.csrf().disable();

        http.exceptionHandling()
                .authenticationEntryPoint(customAuthenticationEntryPoint);

        http.formLogin()
                .failureHandler(new SimpleUrlAuthenticationFailureHandler())
                .successHandler(new CustomAuthenticationSuccessHandler());

        http.authorizeRequests()
                .antMatchers(HttpMethod.POST, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.PUT, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.DELETE, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.GET, "/**").permitAll()
                .anyRequest().denyAll();

    }
[...]
```

E assim finalizamos a implementação de segurança feita de maneira manual e com `Spring Security`,
baseada em sessão.

Essa implementação tem um problema, no entanto: por ser baseada em sessão, ela impede que escalemos
horizontalmente nosso servidor (se o motivo disso não estiver claro para o leitor, sugiro a
pesquisa sobre o conceito de *sticky sessions*). Para fugir disso temos que de alguma forma
não usar a sessão para autenticação, mas isso complica um pouco nossa vida. O primeiro passo
nesse nosso objetivo é desligar a sessão de vez, vamos fazer isso na classe
`SegurancaConfiguration`:

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        // em APIs de dados não usamos CSRF
        http.csrf().disable();

        http.exceptionHandling()
                .authenticationEntryPoint(customAuthenticationEntryPoint);

        http.sessionManagement()
                .disable();

        /*http.formLogin()
                .failureHandler(new SimpleUrlAuthenticationFailureHandler())
                .successHandler(new CustomAuthenticationSuccessHandler());*/

        http.authorizeRequests()
                .antMatchers(HttpMethod.POST, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.PUT, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.DELETE, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.GET, "/**").permitAll()
                .anyRequest().denyAll();

    }
```

Repare que também desativamos o `formLogin`, já que ele não nos ajuda mais pois está acoplado
ao armazenamento de usuário na sessão. Mas e agora o que colocaremos no lugar? Utilizaremos
`Basic Authentication`. Essa autenticação consiste basicamente na informação de credenciais
em todas as requisições, através de um header especial que é o `Authorization`. Vamos ver
como isso funciona:

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        // em APIs de dados não usamos CSRF
        http.csrf().disable();

        http.exceptionHandling()
                .authenticationEntryPoint(customAuthenticationEntryPoint);

        http.sessionManagement()
                .disable();

        /*http.formLogin()
                .failureHandler(new SimpleUrlAuthenticationFailureHandler())
                .successHandler(new CustomAuthenticationSuccessHandler());*/

        http.httpBasic()
                .authenticationEntryPoint(customAuthenticationEntryPoint);

        http.authorizeRequests()
                .antMatchers(HttpMethod.POST, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.PUT, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.DELETE, "/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.GET, "/**").permitAll()
                .anyRequest().denyAll();

    }
```

Suba a aplicação e teste agora uma requisição protegida enviando o header `Authorization`
com as credenciais (usuário e senha, isso é possível usando o tipo `Basic`). No Postman existe
uma aba chamada `Authorization` que fornece uma maneira fácil de fazer isso.

Já ficou bem melhor, mas não me parece muito prático e seguro armazenar o login e senha do
usuário durante *todo* o tempo em que ele estiver usando a aplicação, certo? Nessa hora que
entram os chamados *tokens de autenticação*, que são chaves geradas pelo servidor que permitem
aferir posteriormente que uma autenticação com sucesso ocorreu no passado.

Vamos começar criando uma entidade chamada `TokenAutenticacao` no pacote de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.ManyToOne;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
public class TokenAutenticacao {

    @Id
    private UUID id;

    @ManyToOne
    private Usuario usuario;

    private LocalDateTime validoAte;

    // getters e setters omitidos
}
```

E adicionar o changeset liquibase responsável pela criação da tabela. Primeiro no arquivo
`db.changelog-master.yml`:

```yaml
  - changeSet:
      id: 4
      author: samuelgrigolato
      changes:
      - sqlFile:
          dbms: postgresql
          encoding: utf8
          path: sqls/0004-cria-tabela-tokens-autenticacao.sql
          relativeToChangelogFile: true
```

E depois o SQL correspondente:

```sql
create table token_autenticacao (
    id uuid not null,
    usuario_id uuid not null,
    valido_ate timestamp with time zone,

    constraint pk_tokenautenticacao primary key (id),
    constraint fk_tokenautenticacao_usuario foreign key (usuario_id)
      references usuario (id)
);
```

Agora que temos a entidade, vamos criar um repositório para ela (interface
`TokensAutenticacaoRepository` no pacote de segurança):

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.data.repository.CrudRepository;

import java.util.UUID;

public interface TokensAutenticacaoRepository extends CrudRepository<TokenAutenticacao, UUID> {
}
```

A ideia é não usarmos mais o tipo de autenticação `Basic`, mas o `Bearer`, cujo intuito é
justamente receber chaves que o servidor consegue processar e aceitar ou não, autenticando
o interessado.

Um problema que vamos notar ao tentar usar um header de autorização do tipo `Bearer` é que
o Spring Security *não possui suporte para ele* (diferentemente do tipo `Basic` como
vimos), então para adicionar esse suporte temos que implementar um filtro customizado.
Vamos começar criando a classe `BearerTokenAuthenticationFilter` no pacote de segurança:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.GenericFilterBean;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.UUID;

@Component
public class BearerTokenAuthenticationFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        if (request instanceof HttpServletRequest) {
            String authorization = ((HttpServletRequest) request).getHeader("Authorization");
            if (authorization != null && authorization.startsWith("Bearer ")) {
                String tokenStr = authorization.substring("Bearer ".length());
                UUID token = UUID.fromString(tokenStr);
                SecurityContextHolder.getContext().setAuthentication(new BearerTokenAuthentication(token));
            }
        }

        chain.doFilter(request, response);

    }

}
```

E também a classe `BearerTokenAuthentication` no mesmo pacote:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;

import java.util.List;
import java.util.UUID;

public class BearerTokenAuthentication extends AbstractAuthenticationToken {

    private UUID token;

    public BearerTokenAuthentication(UUID token) {
        super(null);
        this.token = token;
    }

    public BearerTokenAuthentication(UUID token, List<GrantedAuthority> authorities) {
        super(authorities);
        this.token = token;
        setAuthenticated(true);
    }

    @Override
    public Object getCredentials() {
        return token;
    }

    @Override
    public Object getPrincipal() {
        return token;
    }
}

```

Por fim vamos registrar este filtro na classe `SegurancaConfiguration`:

```java
/*http.formLogin()
        .failureHandler(new SimpleUrlAuthenticationFailureHandler())
        .successHandler(new CustomAuthenticationSuccessHandler());*/

/*http.httpBasic()
        .authenticationEntryPoint(customAuthenticationEntryPoint);*/

http.addFilterBefore(new BearerTokenAuthenticationFilter(), BasicAuthenticationFilter.class);
```

Agora podemos adaptar a classe `CustomAuthenticationProvider` de modo que ela suporte tokens
do tipo `BearerTokenAuthentication`, e tente buscar na base de dados um token existente:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.Arrays;
import java.util.UUID;

@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private TokensAutenticacaoRepository tokensAutenticacaoRepository;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        BearerTokenAuthentication bearerToken = (BearerTokenAuthentication) authentication;
        UUID token = (UUID)bearerToken.getCredentials();

        TokenAutenticacao tokenAutenticacao = tokensAutenticacaoRepository.findOne(token);

        if (tokenAutenticacao == null) {
            throw new BadCredentialsException("Token inválido");
        }

        if (tokenAutenticacao.getValidoAte().isBefore(LocalDateTime.now())) {
            throw new BadCredentialsException("Token expirado");
        }

        return new BearerTokenAuthentication(token,
                Arrays.asList(new SimpleGrantedAuthority("ADMIN")));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.isAssignableFrom(BearerTokenAuthentication.class);
    }

}
```

Agora podemos testar, para isso vamos inserir na mão um token (ajuste os IDs se necessário):

```sql
insert into token_autenticacao (id, usuario_id, valido_ate)
values ('8ace3001-05f3-4a25-a776-5f9eacd0b518', 'f8fcc35b-5515-4e06-9c47-7332c9494918', '2018-01-01');
```

E agora a única coisa que falta é fornecermos um endpoint que permite a obtenção desses tokens!

Para isso, vamos voltar com nossa classe `LoginAPI` no pacote de segurança, mas agora com uma
cara diferente:

```java
package com.opensanca.trilharest.filmes.seguranca;

import com.opensanca.trilharest.filmes.comum.BindingException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;
import java.time.LocalDateTime;
import java.util.UUID;

@RestController
@RequestMapping("/login")
public class LoginAPI {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private TokensAutenticacaoRepository tokensAutenticacaoRepository;

    @PostMapping
    public UUID login(@Valid @RequestBody CredenciaisDTO credenciaisDTO, BindingResult results) {
        if (results.hasErrors()) {
            throw new BindingException(results);
        }
        Usuario usuario = usuariosRepository.findByUsuario(credenciaisDTO.getUsuario())
                .orElseThrow(CredencialInvalidaException::new);
        if (!passwordEncoder.matches(credenciaisDTO.getSenha(), usuario.getSenha())) {
            throw new CredencialInvalidaException();
        }
        UUID tokenId = UUID.randomUUID();
        TokenAutenticacao token = new TokenAutenticacao();
        token.setId(tokenId);
        token.setUsuario(usuario);
        token.setValidoAte(LocalDateTime.now().plusMinutes(30));
        tokensAutenticacaoRepository.save(token);
        return tokenId;
    }

}
```

Por fim temos que liberar o uso desse endpoint para todos os usuários independente de estarem
ou não autenticados. Fazemos isso no `SegurancaConfiguration`:

```java
http.authorizeRequests()
        .antMatchers(HttpMethod.POST, "/login").permitAll()
        .antMatchers(HttpMethod.POST, "/**").hasAuthority("ADMIN")
        .antMatchers(HttpMethod.PUT, "/**").hasAuthority("ADMIN")
        .antMatchers(HttpMethod.DELETE, "/**").hasAuthority("ADMIN")
        .antMatchers(HttpMethod.GET, "/**").permitAll()
        .anyRequest().denyAll();
```

Agora conseguimos obter tokens através do endpoint `POST /login` e usá-los como header
`Authentication` do tipo `Bearer`. Essa estratégia é bem fácil de ser integrada com frameworks
de frontend como Angular, React etc.

Antes de passarmos para o próximo tópico, como saber dentro de uma action qual o usuário logado
(por exemplo para vinculá-lo como usuário que cadastrou ou alterou um registro)? Isso é bem
simples, basta anotarmos um parâmetro de action com a anotação `@AuthenticationPrincipal`, se
esse parâmetro for compatível com o objeto retornado pelo método `getPrincipal()` do nosso
`BearerTokenAuthentication` (no nosso caso o parâmetro deve ser do tipo UUID) o Spring vai
injetar o parâmetro na action sozinho, veja:

```java
[...]

@RestController
@RequestMapping("/filmes")
@Api("API de filmes em cartaz")
public class FilmesAPI {

[...]

  @PostMapping
  public UUID cadastrar(@Valid @RequestBody FilmeFormDTO dados, BindingResult results, @AuthenticationPrincipal UUID token) {

    System.out.println("Cadastro solicitado pelo token " + token);

    validarNomeDuplicado(dados, null, results);
    if (results.hasErrors()) {
      throw new BindingException(results);
    }
    Filme entidade = dados.construir();
    filmesRepository.save(entidade);
    return entidade.getId();
  }

[...]
```

E agora para o último tópico antes de encerrar o assunto de autenticação/autorização: `JSON Web
Tokens`. Não seria interessante não termos que ter uma tabela de tokens na nossa base de dados?
Ela só vai crescer com o tempo, gera várias consultas a cada requisição, dentre outros problemas.
Será que existe alguma solução? Ocorre que sim, existe, e essa solução é o uso de JWT. JWT é
basicamente uma maneira de entregar um token criptografado de uma maneira que **apenas** o
servidor consiga descriptografá-lo (isso é diferente de espalhamento, com esses algoritmos
há sim uma maneira de se chegar na informação original). Esse conceito é fortemente baseado em
criptografia de chaves assimétricas, caso o leitor não conheça sobre o assunto sugiro que adicione
na lista de coisas a estudar.

A ideia é, no endpoint `POST /login`, gerar um JWT ao invés de salvar um registro no banco, sendo
que neste JWT estarão contidos o ID do usuário que se autenticou com sucess, bem como uma data
de validade. Já no nosso `CustomAuthenticationProvider`, ao invés de ir buscar um token no banco
de dados, iremos descriptografar o JWT recebido e confiar na informação (o que vai nos dar
muita performance e simplicidade).

Vamos começar então adicionando uma biblioteca para manipulação de JWTs, no nosso `build.gradle`:

```groovy
compile group: 'io.jsonwebtoken', name: 'jjwt', version: '0.9.0'
```

E então adaptar a classe `LoginAPI` para gerar um JWT ao invés de um registro no banco:

```java
package com.opensanca.trilharest.filmes.seguranca;

import com.opensanca.trilharest.filmes.comum.BindingException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;

@RestController
@RequestMapping("/login")
public class LoginAPI {

    @Autowired
    private UsuariosRepository usuariosRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private TokensAutenticacaoRepository tokensAutenticacaoRepository;

    @Value("${autenticacao.secret}")
    private String secret;

    @Value("${autenticacao.jwtSecret}")
    private String jwtSecret;

    @PostMapping
    public String login(@Valid @RequestBody CredenciaisDTO credenciaisDTO, BindingResult results) {
        if (results.hasErrors()) {
            throw new BindingException(results);
        }
        Usuario usuario = usuariosRepository.findByUsuario(credenciaisDTO.getUsuario())
                .orElseThrow(CredencialInvalidaException::new);
        if (!passwordEncoder.matches(credenciaisDTO.getSenha(), usuario.getSenha())) {
            throw new CredencialInvalidaException();
        }

        /*UUID tokenId = UUID.randomUUID();
        TokenAutenticacao token = new TokenAutenticacao();
        token.setId(tokenId);
        token.setUsuario(usuario);
        token.setValidoAte(LocalDateTime.now().plusMinutes(30));
        tokensAutenticacaoRepository.save(token);*/

        return Jwts.builder()
                .setSubject(usuario.getUsuario())
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }

}
```

E agora para a parte de validação, primeiro na classe `BearerAuthenticationToken`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;

import java.util.List;

public class BearerTokenAuthentication extends AbstractAuthenticationToken {

    private String jwt;

    public BearerTokenAuthentication(String jwt) {
        super(null);
        this.jwt = jwt;
    }

    public BearerTokenAuthentication(String jwt, List<GrantedAuthority> authorities) {
        super(authorities);
        this.jwt = jwt;
        setAuthenticated(true);
    }

    @Override
    public Object getCredentials() {
        return jwt;
    }

    @Override
    public Object getPrincipal() {
        return jwt;
    }
}
```

Na classe `BearerTokenAuthenticationFilter`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.GenericFilterBean;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

@Component
public class BearerTokenAuthenticationFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        if (request instanceof HttpServletRequest) {
            String authorization = ((HttpServletRequest) request).getHeader("Authorization");
            if (authorization != null && authorization.startsWith("Bearer ")) {
                String jwt = authorization.substring("Bearer ".length());
                SecurityContextHolder.getContext().setAuthentication(new BearerTokenAuthentication(jwt));
            }
        }

        chain.doFilter(request, response);

    }

}
```

E por fim na classe `CustomAuthenticationProvider`:

```java
package com.opensanca.trilharest.filmes.seguranca;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private TokensAutenticacaoRepository tokensAutenticacaoRepository;

    @Value("${autenticacao.jwtSecret}")
    private String jwtSecret;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        BearerTokenAuthentication bearerToken = (BearerTokenAuthentication) authentication;
        String jwt = (String)bearerToken.getCredentials();

        try {

            String usuario = Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(jwt)
                .getBody()
                .getSubject();
            System.out.println(usuario + " autenticado com sucesso!");

            return new BearerTokenAuthentication(jwt,
                    Arrays.asList(new SimpleGrantedAuthority("ADMIN")));
        } catch (SignatureException ex) {
            throw new BadCredentialsException("JWT inválido");
        }
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.isAssignableFrom(BearerTokenAuthentication.class);
    }

}
```

Antes de testar, temos que adicionar a propriedade `autenticacao.jwtSecret` no arquivo
`application.properties` colocando uma representação Base64 de uma chave que pode ser
gerada com o seguinte código (isso pode ser executado em uma sessão de debug ou em um
*main*):

```java
java.util.Base64.getEncoder().encodeToString(
    io.jsonwebtoken.impl.crypto.MacProvider.generateKey().getEncoded())
```

```
autenticacao.jwtSecret=/74cfVXgvlbGnsuClrFSfyOemI9t2wz32ZDRjuJ0p0AU5VjwRJqgOIUOvabPsBELhp9ENFHGEoNNuXxhlRvAeg==
```

Pronto, agora temos uma solução usando Spring Security junto com JSON Web Tokens, depois de
passar por soluções convencionais usando Cookies e sessão bem como tokens armazenados no banco
de dados.
