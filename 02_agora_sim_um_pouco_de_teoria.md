# Agora sim vamos falar um pouco sobre teoria

Este tópico agrega os conceitos que foram discutidos no primeiro dia da trilha.
Encoraja-se o leitor a se aprofundar nos termos que lhe são estranhos.

* Conceitos fundamentais da web:
  * Domain Name System (DNS)
  * Protocolo HTTP (requisições e respostas)
    * Path
    * Métodos (GET, POST, PUT, DELETE, PATCH, ...)
    * Headers
    * Cookies
  * HTML
  * CSS
  * JavaScript: atenção, JavaScript é um poço sem fundo de material, aprecie com moderação
e não se perca do objetivo principal que é o domínio de APIs RESTful.

Existe um ótimo livro que aborda detalhadamente essas questões: [Desconstruindo a Web](https://www.casadocodigo.com.br/products/livro-desconstruindo-web).

* *Gerações* de aplicação web ¹:
  1. Servir diretórios de arquivos HTML sem processamento, espelhando 100% um diretório
no sistema de arquivos do servidor. Esse seria o `Apache Httpd` por exemplo servindo
um diretório sem qualquer tipo de processamento intermediário.
  2. Servir conteúdo dinâmico usando alguma linguagem como `PHP` ou `JSP`, mas ainda
assim espelhando um diretório no servidor.
  3. Adoção da arquitetura Model-View-Controller (MVC). Neste ponto perdemos a noção
de sistema de arquivos no servidor, mas o conteúdo das páginas (HTML etc) ainda é
gerado totalmente do lado do servidor.
  4. Adoção de APIs de dados e renderização client-side: com o advento de frameworks como
Angular e React, a implementação de `Single-Page Applications` (os SPAs) se tornaram
bem comuns, consumindo APIs (sejam elas RESTful ou não) no servidor para obter e armazenar
os dados necessários.

¹ Essa terminologia não possui respaldo em literatura, foi apenas uma classificação
criada pelo instrutor com o fim de nortear as discussões.

Estabelecido que focaremos no que chamamos de geração 4 de aplicações web, vamos analisar
uma possível API (que não usa as diretrizes RESTful) para atender a um módulo de cadastro
de clientes:

```
GET /api/listar-clientes # retorna uma lista de clientes
POST /api/cadastrar-cliente # insere um novo cliente no banco
GET /api/dados-do-cliente?id=1 # retorna os dados completos de um cliente específico
POST /api/alterar-cliente?id=1 # altera os dados de um cliente
POST /api/deletar-cliente?id=1 # exclui um cliente
```

Vamos agora aplicar o principal conceito REST nesta API, o conceito de *recurso*, e ver
como ela fica:

```
GET /clientes
POST /clientes
GET /cliente/5
PUT /clientes/5
DELETE /clientes/5
```

Note que a combinação do caminho com o método HTTP já deixa muito claro a razão de
existência do endpoint, e é isso que buscamos. Um outro benefício é a naturalidade com
que esse tipo de API se expande conforme a necessidade muda. Veja como poderíamos
mapear endpoints para recuperar todos os telefones de um cliente (ao invés de todos
os dados) ou de adicionar um e-mail alternativo sem impactar o restante do registro:

```
GET /clientes/10/telefones
POST /clientes/10/emails
```

Precisa remover um e-mail? Sem problemas:

```
DELETE /clientes/10/emails/2
```

Ter uma API RESTful não é só isso (não falamos nada de relações entre recursos por
exemplo) mas seguir essa diretriz centrada nos *recursos* enquanto desenha sua API
já te dá boa parte dos benefícios.

Exercício para o leitor: como você modelaria um endpoint para uma transferência bancária
de maneira RESTful? Dica: a chave é encontrar o *recurso* REST, ele não é a conta de origem nem a conta de destino, mas sim a concretização do conceito abstração de transferência, ou seja, um endpoint candidato seria `POST /transferencias`. Sem querer
ganhamos facilidades inesperadas no futuro, quando quisermos `GET /transferencias/10/situacao` ou `DELETE /transferencias/5` (para solicitar um cancelamento).

Aqui finalizamos a parte teórica inicial.
[Clique aqui](03_hora_de_especificar.md) para ir para o próximo tópico.

