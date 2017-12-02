# Análise de performance usando jMeter

Uma parte bem importante nos testes de software é garantir que o sistema irá funcionar com
o número de usuários previstos para ele. Por exemplo, se nossa aplicação está prevista para atender 100 usuários simultâneos na página de filmes em exibição, como saber se ela
vai resistir à demanda? Esse tipo de teste costuma ser bom em identificar gargalos em
algoritmos mal implementados, consultas SQL não otimizadas, e também infraestrutura fraca.

O jMeter é uma das mais tradicionais ferramentas para nos ajudar com esses testes de
performance. Vamos começar efetuando o download dele aqui: http://jmeter.apache.org/.

Depois de efetuar o download e descompactar, execute o binário dentro da pasta `bin`
relativa à extração.

O jMeter abre com um plano de teste em branco. Comece preenchendo o campo `Nome` do
plano de teste e salvando o arquivo.

Vamos efetuar um teste básico, onde executaremos uma quantidade grande de requisições
contra um endpoint de acesso anônimo. Para isso, comece clicando com o botão direito
no nó raíz do plano de teste e adicione um `Thread (Users) -> Grupo de Usuários`.
Configure o grupo de usuários da seguinte forma:

* Número de usuários virtuais: 100
* Contador de iteração: 25

Dentro do grupo de usuários, crie um elemento do tipo `Testadores -> Requisição HTTP`.
Configure-o assim:

* Nome do servidor ou IP: localhost
* Número da porta: 8080
* Caminho: /filmes/

Ainda no elemento do grupo de usuários, adicione agora três `Ouvintes`:

* Ver Árvore de Resultados
* Gráfico Agregado
* Gráfico de Resultados

Não é necessário mudar nada nos ouvintes. Agora deixe a aplicação **desligada** e clique
no botão `Play` no menu do jMeter.

Podemos analisar a saída dos três ouvintes para ver como o jMeter se comportou neste
cenário. Agora ligue a aplicação, acione o botão `Limpar Tudo` no menu e depois aperte
`Play` novamente. Veja agora a diferença nos ouvintes.

Para deixar o gráfico um pouco mais interessante podemos simular alguns gargalos. Por
exemplo, vamos deixar uma em cada 50 requisições com 3 segundos de atraso (altere
o conteúdo do método `getTodos` na classe `FilmesAPI` por este que segue):

```java
  @GetMapping
  public Page<FilmeResumidoDTO> getTodos(Pageable parametrosDePaginacao) {

    if (System.currentTimeMillis() % 50 == 0) {
      try {
        Thread.sleep(3000);
      } catch (InterruptedException e) {
      }
    }

    if (parametrosDePaginacao == null) {
      parametrosDePaginacao = new PageRequest(1, 3);
    }
    return filmesRepository.buscarPagina(parametrosDePaginacao);
  }
```

Rode a aplicação novamente, limpe o resultado agregado do jMeter e execute o plano. Veja
agora como ficaram os resultados.

E assim finalizamos todo o material da trilha. Espero que tenha aproveitado, qualquer
coisa você pode encontrar o autor no Slack do OpenSanca ou por e-mail em
samuelgrigolato (at) gmail (dot) com.
